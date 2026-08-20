# Apostila InterSystems ObjectScript Specialist
## Capítulo 8 — T3.1 Differentiates Storage Media (Meios de armazenamento)

> Começa aqui o domínio **T3 — IRIS Features** (14 questões de 76). Este capítulo abre a caixa preta: você vai ver **onde** os dados realmente moram, quais são os tipos de armazenamento disponíveis e como escolher o certo para cada situação.

---

## 1. Objetivo do capítulo

Ao terminar este capítulo, você será capaz de:

1. Distinguir os quatro meios de armazenamento do IRIS: **variável local**, **global privada do processo (PPG)**, **global temporária** e **global persistente**.
2. Explicar o que é um **nó**, o que são **subscritos** e como uma global é uma árvore.
3. Usar os comandos e funções de navegação: **`$DATA`**, **`$GET`**, **`$ORDER`**, **`$QUERY`**, **`MERGE`** e **`KILL`**.
4. Interpretar os quatro valores de retorno de **`$DATA`**: `0`, `1`, `10` e `11`.
5. Abrir a caixa preta de uma classe persistente e ler as globais **`^Pacote.ClasseD`**, **`^Pacote.ClasseI`** e **`^Pacote.ClasseS`**.
6. Entender a **definição de armazenamento** (`Storage`) gerada pelo compilador e o que cada localização significa.
7. Usar **referências globais estendidas** para acessar dados de outro namespace ou de outra base.
8. Explicar o que é **mapeamento** de globais, rotinas e pacotes, e para que serve.
9. Saber o que é **journalizado** e o que não é, e as consequências disso.
10. Escolher conscientemente o meio de armazenamento certo para cada necessidade.
11. Evoluir o projeto: cache de relatório em PPG, área de preparação de importação em global temporária e um inspetor de armazenamento.

---

## 2. O conceito em linguagem de gente

### 2.1 Quatro lugares para guardar coisas

Volte ao laboratório. Você tem quatro lugares onde pode anotar informação, e escolher errado custa caro.

**1. O papelzinho na sua mão — a variável local.**

Você anota um número para usar daqui a dez segundos, e depois amassa e joga fora. Some quando você sai da sala. Ninguém mais vê. É rapidíssimo de escrever e de ler.

No IRIS: `SET total = 100`. Vive na memória do processo, morre com ele, invisível para os outros.

**2. A prancheta pendurada na sua mesa — a global privada do processo (PPG).**

Maior que o papelzinho: cabe uma tabela inteira. Continua sendo **só sua** — ninguém mais enxerga —, e some quando você vai embora no fim do expediente. Mas, diferentemente do papelzinho, ela não ocupa espaço na sua mão: fica pendurada na parede e você consulta quando precisa.

No IRIS: `SET ^||Buffer("linha", 1) = "dados"`. Aquele **duplo pipe** (`||`) é o que a torna privada do processo. Ela vive num banco temporário, é invisível para outros processos e **é apagada automaticamente** quando o processo termina.

**3. O quadro branco da sala de reuniões — a global temporária.**

Todo mundo vê e escreve. Mas é apagado quando o prédio fecha para manutenção, e ninguém espera que o conteúdo dure. Serve para trabalho coletivo de curta duração.

No IRIS: globais com nomes reservados que ficam num banco temporário. Compartilhadas entre processos, **não journalizadas**, e limpas quando a instância reinicia.

**4. O arquivo de aço — a global persistente.**

Fica para sempre. Todo mundo vê. Vai para o backup. É desfeita por rollback. É onde os dados de verdade moram.

No IRIS: `SET ^Patient(1, "name") = "Maria"`. É o armazenamento definitivo.

Resumindo numa tabela que vale a pena decorar:

| Meio | Sintaxe | Quem vê | Quanto dura | Journal | Custo |
|---|---|---|---|---|---|
| Variável local | `x` | só o processo | até o fim do bloco/processo | não | mais barato |
| PPG | `^||x` | só o processo | até o fim do processo | não | barato |
| Global temporária | `^IRIS.TempAlgo` | todos | até reiniciar a instância | não | médio |
| Global persistente | `^x` | todos | para sempre | sim | mais caro |

A regra de escolha, em uma frase: **use o meio menos duradouro que resolva o seu problema.** Guardar em disco o que morre em dois segundos é desperdício; guardar na memória o que precisa sobreviver é perda de dados.

### 2.2 A global é uma árvore, não uma tabela

Este é o modelo mental mais importante do capítulo.

Uma global não é uma tabela com linhas e colunas. É uma **árvore de gavetas dentro de gavetas**, e cada gaveta pode conter tanto um conteúdo próprio quanto outras gavetas.

Pense num arquivo físico organizado assim:

```
Pacientes
├── 1
│   ├── nome    → "Maria Silva"
│   ├── nasc    → "1990-05-17"
│   └── exames
│       ├── 1   → "HGB 13.5"
│       └── 2   → "GLU 92"
└── 2
    ├── nome    → "Bruno Lima"
    └── nasc    → "1978-11-03"
```

No IRIS, isso se escreve assim:

```objectscript
set ^Pacientes(1, "nome") = "Maria Silva"
set ^Pacientes(1, "nasc") = "1990-05-17"
set ^Pacientes(1, "exames", 1) = "HGB 13.5"
set ^Pacientes(1, "exames", 2) = "GLU 92"
set ^Pacientes(2, "nome") = "Bruno Lima"
set ^Pacientes(2, "nasc") = "1978-11-03"
```

Vocabulário que você precisa dominar:

- **Nó** (*node*) — cada gaveta. `^Pacientes(1, "nome")` é um nó.
- **Subscritos** (*subscripts*) — o que vai entre parênteses. `1` e `"nome"` são os subscritos daquele nó.
- **Nível** — a profundidade. `^Pacientes(1)` está no nível 1; `^Pacientes(1, "nome")` está no nível 2.
- **Nó raiz** — a global sem subscrito nenhum: `^Pacientes`.

E três características que surpreendem quem vem de outros modelos:

**a) Um nó pode ter valor E filhos ao mesmo tempo.**

```objectscript
set ^Log = 3                     // a raiz tem valor
set ^Log(1) = "primeira entrada" // e também tem filhos
```

Isso é comum e útil: você viu no Capítulo 5 o `$INCREMENT(^BankLog)` guardando o contador na raiz e as entradas nos filhos.

**b) Os subscritos não precisam ser declarados.**

Não existe esquema. Você grava `^X("a")` hoje e `^X(1, 2, 3, "z")` amanhã, na mesma global, sem avisar ninguém.

**c) Os subscritos vêm sempre ordenados.**

O IRIS mantém a árvore ordenada automaticamente. Números vêm antes de texto, e cada grupo em ordem crescente. Isso é o que faz a navegação com `$ORDER` ser rápida — e é a base de como os índices funcionam.

### 2.3 A caixa preta aberta: onde a sua classe mora

Aqui está a revelação do capítulo.

Quando você compila `LabStudy.Patient` como `%Persistent`, o IRIS cria automaticamente uma **definição de armazenamento** e escolhe globais para guardar tudo. Por convenção, elas são:

- **`^LabStudy.PatientD`** — o **D** de *data*: os dados dos objetos.
- **`^LabStudy.PatientI`** — o **I** de *index*: os índices.
- **`^LabStudy.PatientS`** — o **S** de *stream*: o conteúdo dos streams.

E a estrutura dentro delas é previsível:

```
^LabStudy.PatientD              = 7                     ← contador do último ID
^LabStudy.PatientD(1)           = <lista de valores>    ← o objeto de ID 1
^LabStudy.PatientD(2)           = <lista de valores>    ← o objeto de ID 2

^LabStudy.PatientI("NameIdx", "MARIA SILVA", 1) = ""    ← índice: valor → ID
^LabStudy.PatientI("RecordIdx", "REG-000001", 1) = ""
```

Repare em duas coisas que explicam muito do que você aprendeu antes:

1. **A raiz da global de dados guarda o contador de IDs.** É por isso que, no Capítulo 5, o rollback desfez o paciente mas o número da sequência seguiu adiante.
2. **O índice guarda o valor como subscrito e o ID no fim, com o nó valendo vazio.** Toda a informação está na *posição*, não no conteúdo. É por isso que buscar por índice é rápido: o IRIS pula direto para o subscrito, sem varrer nada.

Essa é a razão do slogan do IRIS: objetos, SQL e globais **não são três bancos**. São três portas para a mesma árvore.

### 2.4 O que é mapeamento

Você aprendeu no Capítulo 0 que o namespace é a "sala" e o database é o "armário". O **mapeamento** é a instrução que diz: *"nesta sala, a gaveta com esta etiqueta fica naquele armário lá do outro corredor"*.

Isso permite coisas muito úteis:

- **Global mapping** — a global `^Config` do namespace `LABSTUDY` fica fisicamente num banco compartilhado, e outros namespaces enxergam exatamente a mesma `^Config`. Uma tabela de configuração única para vários sistemas.
- **Routine mapping** — o mesmo, para código.
- **Package mapping** — o mesmo, para um pacote inteiro de classes. É assim que classes utilitárias comuns ficam num único lugar e são vistas por todos os namespaces.

E há um uso especialmente relevante para este capítulo: mapear certas globais para um banco **sem journaling** ou **temporário**, quando o conteúdo delas não precisa sobreviver nem ser recuperado.

Mapeamentos são configurados no Portal, em **System Administration → Configuration → System Configuration → Namespaces → Global Mappings**.

---

## 3. A sintaxe explicada

### 3.1 Escrevendo e apagando

```objectscript
set ^Data("a") = 1
set ^Data("a", "x") = 10
set ^Data("b") = 2

kill ^Data("a", "x")     // apaga um nó e todos os seus descendentes
kill ^Data                // apaga a global inteira
```

**`KILL` apaga o nó indicado e toda a subárvore abaixo dele.** Isso vale ouro e é fonte de acidentes: `KILL ^Data("a")` remove também `^Data("a","x")`, `^Data("a","y")` e tudo o que houver ali embaixo.

Para apagar apenas o valor de um nó, mantendo os filhos, existe **`ZKILL`** (também escrito `ZK`):

```objectscript
zkill ^Data("a")     // remove o valor de ^Data("a"), preserva os filhos
```

### 3.2 `$DATA`: o nó existe? tem filhos?

**`$DATA(referência)`** responde às duas perguntas de uma vez, devolvendo um de quatro valores:

| Valor | Significado |
|---|---|
| **0** | o nó não existe e não tem descendentes |
| **1** | o nó tem valor e **não** tem descendentes |
| **10** | o nó **não** tem valor, mas **tem** descendentes |
| **11** | o nó tem valor **e** tem descendentes |

Decore lendo assim: **o dígito das dezenas indica filhos; o das unidades indica valor.**

```
LABSTUDY>KILL ^D
LABSTUDY>SET ^D("a") = 1
LABSTUDY>SET ^D("b", "x") = 2
LABSTUDY>SET ^D("c") = 3
LABSTUDY>SET ^D("c", "y") = 4

LABSTUDY>WRITE $DATA(^D("zzz")), !
0
LABSTUDY>WRITE $DATA(^D("a")), !
1
LABSTUDY>WRITE $DATA(^D("b")), !
10
LABSTUDY>WRITE $DATA(^D("c")), !
11
```

Um uso muito comum é como condição:

```objectscript
if $DATA(^Config("timeout")) {
    write ^Config("timeout"), !
}
```

Funciona porque `0` é falso e `1`, `10` e `11` são verdadeiros.

Existe também a forma de dois argumentos, que devolve o valor por referência:

```objectscript
if $DATA(^Config("timeout"), value) {
    write value, !
}
```

### 3.3 `$GET`: leia sem medo

Acessar uma global inexistente causa erro **`<UNDEFINED>`**. Isso derruba o programa.

**`$GET(referência)`** devolve o valor se existir e **string vazia** se não existir — nunca dá erro.

**`$GET(referência, padrão)`** devolve o padrão em vez de vazio.

```
LABSTUDY>WRITE ^D("naoexiste"), !

WRITE ^D("naoexiste")
      ^
<UNDEFINED> *^D("naoexiste")

LABSTUDY>WRITE "[", $GET(^D("naoexiste")), "]", !
[]

LABSTUDY>WRITE $GET(^D("naoexiste"), 0), !
0

LABSTUDY>WRITE $GET(^D("a")), !
1
```

Regra profissional: **em código de produção, leia globais com `$GET`, não diretamente.** A única exceção é quando você acabou de confirmar a existência com `$DATA`.

### 3.4 `$ORDER`: caminhando por um nível

**`$ORDER(referência)`** devolve o **próximo subscrito** existente naquele nível, ou string vazia quando não há mais.

O padrão de laço é este, e você vai escrevê-lo centenas de vezes:

```objectscript
set key = ""
for {
    set key = $ORDER(^D(key))
    quit:key=""
    write key, " = ", $GET(^D(key)), !
}
```

Como funciona:

- Começa-se com `""`, que significa "antes do primeiro".
- Cada chamada avança um subscrito.
- Quando acabam, devolve `""` e o laço termina.

Para percorrer um nível mais profundo, fixe os subscritos de cima:

```objectscript
set id = ""
for {
    set id = $ORDER(^Pacientes(id))
    quit:id=""

    set campo = ""
    for {
        set campo = $ORDER(^Pacientes(id, campo))
        quit:campo=""
        write id, ".", campo, " = ", $GET(^Pacientes(id, campo)), !
    }
}
```

**Percorrendo ao contrário:**

```objectscript
set key = $ORDER(^D(key), -1)
```

O segundo argumento `-1` caminha na **direção inversa**, do fim para o começo. O padrão é `1`.

**Pegando o valor de brinde:**

```objectscript
set key = $ORDER(^D(key), 1, value)
```

O terceiro argumento recebe, por referência, o valor do nó encontrado — evitando uma leitura a mais.

Um detalhe importante: **`$ORDER` não exige que o subscrito atual exista.** Você pode pedir "o próximo depois de `M`" mesmo que não haja nada em `M`. Isso é o que permite buscas por faixa.

### 3.5 `$QUERY`: caminhando pela árvore inteira

Enquanto `$ORDER` anda **num nível**, **`$QUERY`** percorre **toda a árvore**, entrando e saindo dos níveis, na ordem correta.

```objectscript
set ref = "^D"
for {
    set ref = $QUERY(@ref)
    quit:ref=""
    write ref, " = ", @ref, !
}
```

O que ele devolve é uma **string com a referência completa** do próximo nó, algo como `^D("b","x")`.

O sinal **`@`** é o operador de **indireção**: ele diz *"trate o conteúdo desta variável como se fosse código"*. Assim, `@ref` significa "o valor do nó cuja referência está escrita em `ref`". Indireção é um recurso poderoso e será tratado em detalhe no Capítulo 17.

Quando usar cada um:

- **`$ORDER`** — quando você conhece a estrutura e quer percorrer um nível. É o caso normal, e é mais rápido.
- **`$QUERY`** — quando você **não** conhece a estrutura, ou quer despejar tudo. É o que o `ZWRITE` faz por dentro.

### 3.6 `MERGE`: copiando subárvores

```objectscript
merge ^Backup = ^Pacientes
```

**`MERGE`** copia um nó **e toda a sua subárvore** para outro lugar, numa única operação. É muito mais rápido do que percorrer com `$ORDER` e copiar nó a nó.

Funciona entre qualquer combinação de meios:

```objectscript
merge ^||Cache = ^Pacientes(1)      // do disco para a PPG
merge local = ^Config                // da global para uma variável local
merge ^Pacientes(2) = local          // e de volta
```

Dois cuidados:

- `MERGE` **não apaga** o destino antes: ele acrescenta e sobrescreve o que coincidir. Para substituir de verdade, faça `KILL` no destino antes.
- O nó de origem, se tiver valor próprio, é copiado junto.

### 3.7 Globais privadas do processo (PPG)

```objectscript
set ^||Work("line", 1) = "primeira"
set ^||Work("line", 2) = "segunda"

write $GET(^||Work("line", 1)), !

kill ^||Work
```

- O prefixo é **`^||`** — circunflexo seguido de dois pipes.
- Elas se comportam exatamente como globais: subscritos, `$ORDER`, `MERGE`, `$DATA`, tudo funciona.
- São **invisíveis para outros processos**, mesmo com o mesmo nome.
- São **apagadas automaticamente** quando o processo termina.
- **Não são journalizadas** e, portanto, **não são desfeitas por rollback**.

Quando usar PPG em vez de variável local?

- Quando o volume é grande demais para carregar na memória do processo.
- Quando você precisa de **estrutura em árvore com subscritos**, e não só um valor.
- Quando você quer que o dado sobreviva entre chamadas de métodos diferentes, sem passar como argumento.

E o alerta correspondente: como não são journalizadas, **não guarde ali nada que precise participar de uma transação**.

### 3.8 Globais temporárias compartilhadas

O IRIS reserva certos nomes de global para um banco temporário. Globais cujo nome começa com **`^IRIS.Temp`** são, por padrão, mapeadas para o banco temporário da instância:

```objectscript
set ^IRIS.TempLabStudy("import", 1) = "linha crua"
```

Características:

- **Compartilhadas** entre processos.
- **Não journalizadas** — portanto rápidas e não recuperáveis.
- **Apagadas quando a instância reinicia.**

Uso típico: área de trabalho de um processamento em lote que vários processos alimentam, resultados intermediários de um relatório pesado, filas de curta duração.

O conjunto exato de prefixos reservados e o mapeamento padrão podem variar conforme a versão e a configuração da instância: **verificar na documentação oficial** antes de contar com esse comportamento em produção.

### 3.9 Referência global estendida

Por padrão, `^Config` significa "a global `^Config` **deste** namespace". Para acessar a de outro namespace, use a **referência estendida**:

```objectscript
write ^["USER"]Config("x"), !
write ^|"USER"|Config("x"), !
```

As duas formas — colchetes ou pipes — são equivalentes. O namespace pode vir de uma variável:

```objectscript
set ns = "USER"
write ^[ns]Config("x"), !
```

Isso é útil para utilitários que precisam olhar vários namespaces, e é a forma correta de fazê-lo sem ficar trocando de namespace com `ZN` o tempo todo.

Um cuidado: o processo precisa ter **privilégio** sobre o banco do outro namespace, como você viu no capítulo anterior.

### 3.10 A definição de armazenamento da classe

Ao compilar uma classe persistente, o IRIS acrescenta ao final da definição um bloco parecido com este:

```objectscript
Storage Default
{
<Data name="PatientDefaultData">
<Value name="1">
<Value>%%CLASSNAME</Value>
</Value>
<Value name="2">
<Value>Name</Value>
</Value>
<Value name="3">
<Value>BirthDate</Value>
</Value>
</Data>
<DataLocation>^LabStudy.PatientD</DataLocation>
<DefaultData>PatientDefaultData</DefaultData>
<IdLocation>^LabStudy.PatientD</IdLocation>
<IndexLocation>^LabStudy.PatientI</IndexLocation>
<StreamLocation>^LabStudy.PatientS</StreamLocation>
<Type>%Storage.Persistent</Type>
}
```

Traduzindo cada linha importante:

- **`<DataLocation>`** — a global onde os dados dos objetos ficam.
- **`<IdLocation>`** — onde o contador de IDs é mantido (normalmente a raiz da global de dados).
- **`<IndexLocation>`** — a global dos índices.
- **`<StreamLocation>`** — a global do conteúdo dos streams.
- **`<Data>`** e **`<Value>`** — o **mapa de posições**: qual propriedade ocupa qual posição dentro do nó. A posição 1 costuma ser reservada para o nome da classe (usado em herança).

Três consequências práticas:

1. **A ordem das propriedades no armazenamento é fixa depois de compilada.** Acrescentar uma propriedade nova a uma classe com dados a coloca no **fim** da lista, preservando as posições existentes. É por isso que dados antigos continuam legíveis — assunto que retomaremos no Capítulo 11, sobre evolução de esquema.
2. **Você pode editar a definição de armazenamento**, para mapear uma classe sobre uma estrutura de globais já existente. Isso é uma técnica avançada e delicada, usada para modernizar sistemas antigos.
3. **Apagar a classe não apaga a global.** Os dados continuam lá até alguém os remover — como você viu no Capítulo 1.

---

## 4. Exemplo comentado

Uma classe que percorre e compara os meios de armazenamento:

Arquivo `src/LabStudy/Demo/Storage.cls`:

```objectscript
/// Explores the four storage media and the global tree structure.
Class LabStudy.Demo.Storage Extends %RegisteredObject
{

/// Fills the same data structure in all four media.
ClassMethod FillAll() As %Status
{
    kill localVar, ^||PpgData, ^IRIS.TempLabStudy, ^LabStudyDemo

    // 1. local variable: memory of this process, dies with the block
    set localVar("a") = "local value"
    set localVar("a", "child") = "local child"

    // 2. process private global: private, survives the block, dies with the process
    set ^||PpgData("a") = "ppg value"
    set ^||PpgData("a", "child") = "ppg child"

    // 3. shared temporary global: visible to everyone, dies on restart
    set ^IRIS.TempLabStudy("a") = "temp value"
    set ^IRIS.TempLabStudy("a", "child") = "temp child"

    // 4. persistent global: visible to everyone, survives everything
    set ^LabStudyDemo("a") = "global value"
    set ^LabStudyDemo("a", "child") = "global child"

    write "all four media filled", !
    quit $$$OK
}

/// Reports what exists right now in each medium.
ClassMethod ReportAll() As %Status
{
    write "local  : ", $DATA(localVar("a")), " -> [", $GET(localVar("a")), "]", !
    write "ppg    : ", $DATA(^||PpgData("a")), " -> [", $GET(^||PpgData("a")), "]", !
    write "temp   : ", $DATA(^IRIS.TempLabStudy("a")), " -> [", $GET(^IRIS.TempLabStudy("a")), "]", !
    write "global : ", $DATA(^LabStudyDemo("a")), " -> [", $GET(^LabStudyDemo("a")), "]", !
    quit $$$OK
}

/// Walks one level with $ORDER.
ClassMethod WalkLevel(globalName As %String = "^LabStudyDemo") As %Status
{
    write "-- one level of ", globalName, " --", !

    set key = ""
    for {
        set key = $ORDER(@globalName@(key))
        quit:key=""
        write "  ", key, " = [", $GET(@globalName@(key)), "]", !
    }
    quit $$$OK
}

/// Walks the whole tree with $QUERY.
ClassMethod WalkTree(globalName As %String = "^LabStudyDemo") As %Status
{
    write "-- whole tree of ", globalName, " --", !

    set ref = globalName
    for {
        set ref = $QUERY(@ref)
        quit:ref=""
        write "  ", ref, " = [", @ref, "]", !
    }
    quit $$$OK
}

/// Shows the four possible values of $DATA.
ClassMethod ShowData() As %Status
{
    kill ^LabStudyD2

    set ^LabStudyD2("onlyValue") = 1
    set ^LabStudyD2("onlyChild", "x") = 2
    set ^LabStudyD2("both") = 3
    set ^LabStudyD2("both", "y") = 4

    write "missing    : ", $DATA(^LabStudyD2("missing")), !
    write "onlyValue  : ", $DATA(^LabStudyD2("onlyValue")), !
    write "onlyChild  : ", $DATA(^LabStudyD2("onlyChild")), !
    write "both       : ", $DATA(^LabStudyD2("both")), !
    quit $$$OK
}

}
```

Comentando as decisões:

- **`@globalName@(key)`** — esta é a sintaxe de **indireção com subscritos**. `globalName` contém o texto `"^LabStudyDemo"`; o primeiro `@` manda tratar esse texto como referência; o segundo `@`, antes dos parênteses, acrescenta os subscritos. É assim que se escreve código genérico que trabalha sobre qualquer global. Detalhes no Capítulo 17.
- **`kill` no começo de `FillAll`** garante que a demonstração parta sempre do zero. Experimentos que dependem do que sobrou de antes não ensinam nada.
- **`ReportAll` usa `$DATA` e `$GET` juntos**, mostrando os dois lados: existência e conteúdo. Note que a variável local aparecerá como `0` quando `ReportAll` for chamado separadamente de `FillAll` — e essa é a demonstração central do exemplo.

### 4.1 Usando no Terminal

```
LABSTUDY>DO ##class(LabStudy.Demo.Storage).FillAll()
all four media filled

LABSTUDY>DO ##class(LabStudy.Demo.Storage).ReportAll()
local  : 0 -> []
ppg    : 11 -> [ppg value]
temp   : 11 -> [temp value]
global : 11 -> [global value]
```

**Pare aqui e observe o resultado mais importante do capítulo.**

A variável local apareceu como `0`, ou seja, **inexistente** — apesar de ter sido preenchida segundos antes. Por quê? Porque `FillAll` e `ReportAll` são métodos diferentes, e cada método é um bloco de procedimento com variáveis privadas, como você aprendeu no Capítulo 3. A variável local morreu ao sair de `FillAll`.

Os outros três sobreviveram, porque nenhum deles é variável local.

Continuando:

```
LABSTUDY>DO ##class(LabStudy.Demo.Storage).ShowData()
missing    : 0
onlyValue  : 1
onlyChild  : 10
both       : 11

LABSTUDY>DO ##class(LabStudy.Demo.Storage).WalkLevel()
-- one level of ^LabStudyDemo --
  a = [global value]

LABSTUDY>DO ##class(LabStudy.Demo.Storage).WalkTree()
-- whole tree of ^LabStudyDemo --
  ^LabStudyDemo("a") = [global value]
  ^LabStudyDemo("a","child") = [global child]
```

A diferença entre `WalkLevel` e `WalkTree` fica evidente: o primeiro viu **um** nó, o segundo viu **dois**. `$ORDER` anda no nível; `$QUERY` entra nos filhos.

### 4.2 Abrindo a caixa preta da classe persistente

```
LABSTUDY>ZWRITE ^LabStudy.PatientD
^LabStudy.PatientD=1
^LabStudy.PatientD(1)=$lb("","Maria Silva",54683,"REG-000001","F",1,"2026-08-19 15:02:11",$lb("","Potirendaba","SP",""))

LABSTUDY>ZWRITE ^LabStudy.PatientI
^LabStudy.PatientI("NameIdx"," MARIA SILVA",1)=""
^LabStudy.PatientI("RecordIdx","REG-000001")=1
^LabStudy.PatientI("SexIdx","F",1)=""

LABSTUDY>WRITE ^LabStudy.PatientD, !
1
```

Leia com atenção, porque cada linha explica algo que você já usou:

- **`^LabStudy.PatientD` na raiz vale `1`** — é o contador do último ID atribuído. Grave essa informação: é o que garante que IDs não se repitam, e é por isso que apagar objetos não faz o contador voltar.
- **`^LabStudy.PatientD(1)` contém `$lb(...)`** — uma **lista** com os valores das propriedades, na ordem definida no bloco `Storage`. O `$lb` é a forma abreviada de `$LISTBUILD`, o formato interno de lista do IRIS, que veremos em detalhe no Capítulo 9. Repare que a **data de nascimento está lá como `54683`**, o valor lógico que discutimos no Capítulo 2 — a global mostra a verdade nua.
- **O endereço embutido aparece como uma lista dentro da lista.** É exatamente o que o Capítulo 2 prometeu sobre `%SerialObject`: ele é gravado **dentro** do objeto que o contém, sem ID nem global própria.
- **`^LabStudy.PatientI("NameIdx", " MARIA SILVA", 1) = ""`** — o índice normal guarda `nome do índice → valor → ID`, com o nó vazio. Note que o valor foi normalizado para maiúsculas com um espaço à frente: é assim que o IRIS torna a ordenação previsível e a busca insensível a maiúsculas.
- **`^LabStudy.PatientI("RecordIdx", "REG-000001") = 1`** — o índice **único** é diferente: como só pode haver um, o ID vai no **valor** do nó em vez de virar mais um subscrito. É exatamente por isso que o compilador consegue gerar o `RecordIdxOpen()`: basta uma leitura direta, sem varredura.

Nada disso é mágica. É uma árvore, e agora você sabe ler.

---

## 5. Variações e detalhes

### 5.1 Ordenação: como o IRIS ordena subscritos

A ordem que o `$ORDER` segue é definida e vale a pena conhecer:

1. Primeiro o subscrito vazio, se existir.
2. Depois os **valores numéricos canônicos**, em ordem numérica: `1`, `2`, `10`, `100`.
3. Depois os **valores de texto**, em ordem de caractere: `"a"`, `"b"`, `"z"`.

O ponto que confunde: `10` vem **depois** de `2` porque são tratados como números; mas `"10"` (com aspas, tratado como texto) viria **antes** de `"2"`. O IRIS decide pela forma do valor: um número escrito de forma canônica (sem zeros à esquerda, sem sinal desnecessário) é tratado como número.

```
LABSTUDY>KILL ^O
LABSTUDY>SET ^O(2)="", ^O(10)="", ^O("a")="", ^O("B")=""

LABSTUDY>SET k="" FOR { SET k=$ORDER(^O(k)) QUIT:k=""  WRITE k, " " }
2 10 B a
```

Note que `B` veio antes de `a`: a ordenação de texto segue o código dos caracteres, e maiúsculas vêm antes de minúsculas. É por isso que o IRIS normaliza valores nos índices, como você viu na seção 4.2.

### 5.2 Journaling por banco

O journaling é configurado **por banco de dados**, não por global nem por namespace.

Consequências:

- Globais de um banco sem journaling **não são desfeitas por rollback** e **não são recuperáveis** após uma falha.
- É por isso que bancos temporários não têm journaling: o conteúdo é descartável por definição.
- É possível mapear certas globais para um banco sem journaling, deliberadamente, quando o custo do journal não se justifica.

Existem também formas de suspender o journaling dentro de um bloco de código específico. Trata-se de uma técnica de otimização com implicações sérias de integridade, e a forma exata varia por versão: **verificar na documentação oficial** antes de usar.

### 5.3 O tamanho e o custo de uma global

Cada nó de global custa espaço para o valor **e** para os subscritos. Um esquema com subscritos longos e descritivos é mais legível e mais caro:

```objectscript
set ^Log(1, "identificadorDoUsuarioResponsavel") = "ana"     // caro
set ^Log(1, "usr") = "ana"                                     // barato
```

Em volumes pequenos, escreva legível. Em volumes de milhões de nós, os nomes curtos deixam de ser preciosismo e viram diferença de gigabytes.

É possível consultar o tamanho ocupado por uma global no Portal, em **System Explorer → Globals**, que mostra a lista de globais do namespace com seus tamanhos.

### 5.4 Escolhendo o meio: a tabela de decisão

| Preciso de... | Meio |
|---|---|
| um contador dentro de um laço | variável local |
| acumular alguns milhares de linhas durante um processamento, sem compartilhar | PPG (`^||`) |
| uma área de trabalho compartilhada por vários processos, descartável | global temporária |
| dados de negócio, com backup e rollback | global persistente (ou, melhor, uma classe persistente) |
| resultado de um cálculo caro, reaproveitável só nesta sessão | PPG |
| cache compartilhado por todos, reconstruível | global temporária |
| configuração do sistema | global persistente |

E uma recomendação importante que fecha o capítulo: **para dados de negócio, prefira uma classe persistente à global crua.** Você ganha validação, índices, SQL, auditoria por trigger e evolução de esquema — tudo o que os capítulos anteriores construíram. Globais cruas são a ferramenta certa para estruturas auxiliares, contadores, caches e trilhas de altíssimo volume.

### 5.5 Um erro caro: `KILL` sem subscrito

```objectscript
kill ^Pacientes(id)      // apaga um paciente
kill ^Pacientes           // apaga TODOS os pacientes
```

Um subscrito de diferença entre uma operação normal e um desastre. E o `KILL` **não pergunta**.

Duas defesas práticas:

- Nunca escreva `KILL ^Global` num método sem uma verificação explícita antes.
- Se a variável do subscrito puder vir vazia, o `KILL ^Pacientes(id)` com `id` vazio **não** apaga a global inteira — ele apaga o nó de subscrito vazio, o que é diferente. Mas construir a referência por indireção, sem cuidado, pode sim produzir um `KILL` da raiz. Confira sempre.

---

## 6. Pegadinhas e erros comuns

**1) Ler global inexistente diretamente.**
`WRITE ^X("y")` inexistente causa `<UNDEFINED>`. Use `$GET`.

**2) Confundir os valores de `$DATA`.**
`10` significa "sem valor, com filhos"; `11` significa "com valor e com filhos". Dezenas = filhos, unidades = valor.

**3) Achar que `KILL` apaga só o nó.**
Apaga o nó **e toda a subárvore**. Para apagar só o valor, `ZKILL`.

**4) Esquecer o `QUIT:key=""` no laço de `$ORDER`.**
Laço infinito, porque `$ORDER("")` volta ao primeiro subscrito.

**5) Usar `$ORDER` esperando percorrer a árvore inteira.**
`$ORDER` anda **num nível**. Para a árvore toda, `$QUERY`.

**6) Achar que a PPG é journalizada.**
Não é, e portanto não participa de transações nem é desfeita por rollback.

**7) Contar com uma global temporária depois de reiniciar a instância.**
Ela é apagada. É temporária de propósito.

**8) Confundir variável local com PPG.**
`x` morre no fim do método; `^||x` sobrevive até o fim do processo.

**9) Achar que `MERGE` limpa o destino.**
Não limpa: mescla. Faça `KILL` antes se quiser substituir.

**10) Assumir que `"10"` vem depois de `"2"` em texto.**
Em ordenação numérica sim; em ordenação de texto, `"10"` vem antes. O IRIS decide pela forma canônica do valor.

**11) Editar a global de dados de uma classe persistente "na mão".**
Você pode corromper índices e quebrar a consistência. Se fizer, reconstrua com `%BuildIndices()`. Em geral: não faça.

**12) Apagar a classe achando que os dados vão junto.**
Não vão. Use `%KillExtent()` antes.

**13) Achar que `^LabStudy.PatientD` é o nome da tabela SQL.**
Não é. `SqlTableName` afeta o mundo SQL; o nome da global vem do nome da **classe**.

**14) Usar referência estendida sem privilégio no outro banco.**
Falha por segurança, não por sintaxe.

**15) Guardar dados de negócio em globais cruas por hábito.**
Perde validação, índices, SQL e evolução de esquema. Use classe persistente, salvo motivo forte.

---

## 7. MÃO NA MASSA

> Os exercícios 8.1 e 8.4 pedem **duas sessões de Terminal**, para observar o que é privado do processo e o que é compartilhado.

---

### Exercício 8.1 — O tempo de vida de cada meio

**a) Enunciado:** Abra **duas** sessões de Terminal em `LABSTUDY` (A e B).

Na sessão A:
1. Crie uma variável local, uma PPG, uma global temporária e uma global persistente, todas com a mesma estrutura.
2. Confirme com `$DATA` que as quatro existem.

Na sessão B:
3. Verifique quais das quatro a sessão B enxerga.

De volta à sessão A:
4. Encerre a sessão com `HALT` e abra de novo.
5. Verifique de novo quais das quatro sobreviveram.

**b) Dica:** Use `$DATA` para cada uma, imprimindo lado a lado.

**c) Como testar:** A sessão B deve enxergar duas das quatro. Após o `HALT`, a sessão A deve enxergar duas — mas não as mesmas duas do passo 2.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

**Sessão A:**

```
LABSTUDY>SET locVar("x") = "local"
LABSTUDY>SET ^||Priv("x") = "ppg"
LABSTUDY>SET ^IRIS.TempEx81("x") = "temp"
LABSTUDY>SET ^Ex81("x") = "persistent"

LABSTUDY>WRITE $DATA(locVar("x")), $DATA(^||Priv("x")), $DATA(^IRIS.TempEx81("x")), $DATA(^Ex81("x")), !
1111
```

*(Quatro valores `1` colados: as quatro existem.)*

**Sessão B:**

```
LABSTUDY>WRITE $DATA(locVar("x")), " ", $DATA(^||Priv("x")), " ", $DATA(^IRIS.TempEx81("x")), " ", $DATA(^Ex81("x")), !
0 0 1 1
```

A sessão B **não** enxerga a variável local nem a PPG da sessão A. Enxerga a temporária e a persistente.

**Sessão A, depois de `HALT` e reconectar:**

```
LABSTUDY>WRITE $DATA(locVar("x")), " ", $DATA(^||Priv("x")), " ", $DATA(^IRIS.TempEx81("x")), " ", $DATA(^Ex81("x")), !
0 0 1 1
```

**Por que cada decisão:**

- **O experimento tem duas dimensões, e por isso precisa de duas etapas.** A sessão B testa a dimensão **visibilidade**; o `HALT` testa a dimensão **durabilidade**. Um teste só não separaria as duas.
- **O resultado da sessão B é idêntico ao da sessão A reconectada**, e isso não é coincidência: uma nova sessão **é** um novo processo. Local e PPG pertencem ao processo; quando ele acaba, elas acabam.
- **A global temporária sobreviveu ao `HALT`** porque o que a apaga é o reinício da **instância**, não do processo. Se você tiver como reiniciar o IRIS (`docker restart iris-study`), faça o teste e observe: `^IRIS.TempEx81` desaparece e `^Ex81` continua.
- **Um detalhe fácil de perder:** no primeiro `WRITE` da sessão A, os quatro valores saíram colados (`1111`), porque não havia separador. Nas sessões seguintes acrescentei `" "`. Formatar a saída de um experimento faz parte de fazer o experimento direito.

---

### Exercício 8.2 — Navegando numa árvore

**a) Enunciado:** Monte, no Terminal, a global `^Lab` com esta estrutura:

```
^Lab                       = 2                    (contador)
^Lab(1)                    = "Maria Silva"
^Lab(1,"nasc")             = "1990-05-17"
^Lab(1,"exames",1)         = "HGB 13.5"
^Lab(1,"exames",2)         = "GLU 92"
^Lab(2)                    = "Bruno Lima"
^Lab(2,"nasc")             = "1978-11-03"
```

Depois:
1. Mostre `$DATA` de `^Lab`, `^Lab(1)`, `^Lab(1,"nasc")`, `^Lab(1,"exames")` e `^Lab(3)`, explicando cada resultado.
2. Percorra o primeiro nível com `$ORDER`.
3. Percorra os exames do paciente 1.
4. Percorra a árvore inteira com `$QUERY`.
5. Percorra o primeiro nível ao contrário.
6. Copie a subárvore do paciente 1 para `^LabBackup` com `MERGE` e confira.
7. Apague os exames do paciente 1 com um único `KILL`.

**b) Dica:** No item 4, use `$QUERY(@ref)` com `ref` começando em `"^Lab"`.

**c) Como testar:** O `KILL` do item 7 deve remover dois nós com um comando só.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

```
LABSTUDY>KILL ^Lab, ^LabBackup

LABSTUDY>SET ^Lab = 2
LABSTUDY>SET ^Lab(1) = "Maria Silva"
LABSTUDY>SET ^Lab(1,"nasc") = "1990-05-17"
LABSTUDY>SET ^Lab(1,"exames",1) = "HGB 13.5"
LABSTUDY>SET ^Lab(1,"exames",2) = "GLU 92"
LABSTUDY>SET ^Lab(2) = "Bruno Lima"
LABSTUDY>SET ^Lab(2,"nasc") = "1978-11-03"

LABSTUDY>WRITE $DATA(^Lab), !
11
LABSTUDY>WRITE $DATA(^Lab(1)), !
11
LABSTUDY>WRITE $DATA(^Lab(1,"nasc")), !
1
LABSTUDY>WRITE $DATA(^Lab(1,"exames")), !
10
LABSTUDY>WRITE $DATA(^Lab(3)), !
0

LABSTUDY>SET id="" FOR { SET id=$ORDER(^Lab(id)) QUIT:id=""  WRITE id, ": ", ^Lab(id), ! }
1: Maria Silva
2: Bruno Lima

LABSTUDY>SET n="" FOR { SET n=$ORDER(^Lab(1,"exames",n)) QUIT:n=""  WRITE n, " -> ", ^Lab(1,"exames",n), ! }
1 -> HGB 13.5
2 -> GLU 92

LABSTUDY>SET r="^Lab" FOR { SET r=$QUERY(@r) QUIT:r=""  WRITE r, " = ", @r, ! }
^Lab(1) = Maria Silva
^Lab(1,"exames",1) = HGB 13.5
^Lab(1,"exames",2) = GLU 92
^Lab(1,"nasc") = 1990-05-17
^Lab(2) = Bruno Lima
^Lab(2,"nasc") = 1978-11-03

LABSTUDY>SET id="" FOR { SET id=$ORDER(^Lab(id),-1) QUIT:id=""  WRITE id, " " }
2 1

LABSTUDY>MERGE ^LabBackup = ^Lab(1)

LABSTUDY>ZWRITE ^LabBackup
^LabBackup="Maria Silva"
^LabBackup("exames",1)="HGB 13.5"
^LabBackup("exames",2)="GLU 92"
^LabBackup("nasc")="1990-05-17"

LABSTUDY>KILL ^Lab(1,"exames")

LABSTUDY>ZWRITE ^Lab
^Lab=2
^Lab(1)="Maria Silva"
^Lab(1,"nasc")="1990-05-17"
^Lab(2)="Bruno Lima"
^Lab(2,"nasc")="1978-11-03"
```

**Por que cada resultado:**

- **`$DATA(^Lab)` = 11** — a raiz tem valor (o contador `2`) **e** tem filhos. Este é exatamente o caso que a seção 2.2 antecipou, e é o mesmo padrão que o IRIS usa nas globais de classes persistentes.
- **`$DATA(^Lab(1,"exames"))` = 10** — o nó `exames` **não tem valor próprio**; ele existe apenas como caminho para os filhos. Isso é comum: nós intermediários frequentemente são só estrutura.
- **`$DATA(^Lab(3))` = 0** — não existe.
- **Repare na ordem do `$QUERY`:** `^Lab(1,"exames",1)` veio **antes** de `^Lab(1,"nasc")`. Isso porque, no segundo nível do paciente 1, os subscritos são `"exames"` e `"nasc"`, em ordem alfabética. O `$QUERY` desce até o fundo de `"exames"` antes de voltar e seguir para `"nasc"`. É a ordem da árvore, não a ordem em que você gravou.
- **Repare também que o `$QUERY` não mostrou `^Lab` (a raiz).** Ele começa a partir do nó indicado e devolve os **seguintes**.
- **O `MERGE` copiou a subárvore inteira e o valor do nó de origem**, que virou o valor da raiz de `^LabBackup`.
- **Um único `KILL ^Lab(1,"exames")` removeu dois nós.** Não foi preciso percorrer nem apagar um a um. Essa é a força — e o perigo — do `KILL`.

---

### Exercício 8.3 — Abrindo a caixa preta de uma classe

**a) Enunciado:** Usando as classes do projeto:

1. Grave dois pacientes e dois exames.
2. Inspecione `^LabStudy.PatientD` e `^LabStudy.PatientI` com `ZWRITE`.
3. Identifique: onde está o contador de ID? Em que posição da lista está o nome? Como o índice único difere do índice comum?
4. Inspecione `^LabStudy.ExamD` e localize o relacionamento com o paciente.
5. Abra a classe compilada no VS Code e localize o bloco `Storage Default`. Confira se `DataLocation` e `IndexLocation` batem com o que você viu.
6. Apague um paciente e observe o contador.

**b) Dica:** Para ver a definição completa da classe com o bloco `Storage`, abra o arquivo pelo servidor (o bloco é acrescentado na compilação e pode não estar no seu arquivo local até você exportar).

**c) Como testar:** Depois de apagar, o contador na raiz **não** deve diminuir.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

```
LABSTUDY>DO ##class(LabStudy.Patient).%KillExtent()
LABSTUDY>DO ##class(LabStudy.Exam).%KillExtent()
LABSTUDY>KILL ^LabStudySeq

LABSTUDY>SET id1 = ##class(LabStudy.Patient).CreateWithExams("Maria Silva","1990-05-17","F",[{"testCode":"HGB","value":13.5,"unit":"g/dL"}])
LABSTUDY>SET id2 = ##class(LabStudy.Patient).CreateWithExams("Bruno Lima","1978-11-03","M",[{"testCode":"GLU","value":92,"unit":"mg/dL"}])

LABSTUDY>ZWRITE ^LabStudy.PatientD
^LabStudy.PatientD=2
^LabStudy.PatientD(1)=$lb("","Maria Silva",54683,"REG-000001","F",1,"2026-08-19 16:10:02",$lb("","","","",""))
^LabStudy.PatientD(2)=$lb("","Bruno Lima",50539,"REG-000002","M",1,"2026-08-19 16:10:03",$lb("","","","",""))

LABSTUDY>ZWRITE ^LabStudy.PatientI
^LabStudy.PatientI("NameIdx"," BRUNO LIMA",2)=""
^LabStudy.PatientI("NameIdx"," MARIA SILVA",1)=""
^LabStudy.PatientI("RecordIdx","REG-000001")=1
^LabStudy.PatientI("RecordIdx","REG-000002")=2
^LabStudy.PatientI("SexIdx","F",1)=""
^LabStudy.PatientI("SexIdx","M",2)=""

LABSTUDY>ZWRITE ^LabStudy.ExamD
^LabStudy.ExamD=2
^LabStudy.ExamD(1)=$lb("","HGB","2026-08-19 16:10:02",13.5,"g/dL",1,"")
^LabStudy.ExamD(2)=$lb("","GLU","2026-08-19 16:10:03",92,"mg/dL",2,"")

LABSTUDY>WRITE $$$ISOK(##class(LabStudy.Patient).%DeleteId(1)), !
1

LABSTUDY>WRITE ^LabStudy.PatientD, !
2

LABSTUDY>ZWRITE ^LabStudy.PatientD
^LabStudy.PatientD=2
^LabStudy.PatientD(2)=$lb("","Bruno Lima",50539,"REG-000002","M",1,"2026-08-19 16:10:03",$lb("","","","",""))
```

**Por que cada observação importa:**

- **A raiz `^LabStudy.PatientD = 2` é o contador de IDs.** Depois de apagar o paciente 1, ela continuou valendo `2`. Se você criar um paciente novo agora, ele será o ID **3**, não o 1. Isso confirma, olhando os bytes, o que o Capítulo 1 afirmou: **IDs não são reaproveitados**.
- **A primeira posição da lista está vazia (`""`).** Ela é reservada para o nome da classe, usada quando há herança. Como `LabStudy.Patient` não tem subclasse gravando aqui, fica vazia. As propriedades começam na posição 2: `Name`, `BirthDate`, `RecordNumber`, `Sex`, `Active`, `CreatedOn`, e por fim o endereço embutido como uma lista aninhada.
- **A ordem da lista é a ordem de declaração das propriedades na classe.** Isso é o mapa `<Data>`/`<Value>` do bloco `Storage` em ação.
- **`BirthDate` aparece como `54683` e `50539`** — os valores lógicos. Nenhuma conversão acontece no armazenamento. Se você já tinha dúvida sobre "o `%Date` guarda mesmo um número?", aqui está a prova material.
- **O índice comum (`NameIdx`, `SexIdx`) tem o ID como último subscrito e nó vazio.** O índice **único** (`RecordIdx`) tem o ID como **valor**, sem subscrito extra. A razão é lógica: onde só cabe um, não é preciso reservar espaço para vários.
- **Em `^LabStudy.ExamD`, a penúltima posição guarda o ID do paciente** (`1` e `2`). É assim que o lado `one` de um relacionamento é armazenado: uma referência simples ao ID do outro objeto. A lista de exames do lado `many` não é gravada em lugar nenhum — ela é **calculada** consultando o índice sobre essa coluna. Perceber isso explica por que relacionamentos não duplicam dados.

---

### Exercício 8.4 — Referência estendida e visibilidade entre namespaces

**a) Enunciado:**

1. No namespace `USER`, crie a global `^Shared("msg") = "escrito em USER"`.
2. Volte para `LABSTUDY` e tente ler `^Shared("msg")` normalmente. Observe o resultado.
3. Leia com **referência estendida** apontando para `USER`.
4. Escreva, a partir de `LABSTUDY`, um valor novo dentro da global de `USER` usando referência estendida, e confirme em `USER` que chegou.
5. Explique, com suas palavras, por que os passos 2 e 3 dão resultados diferentes.

**b) Dica:** As duas formas de referência estendida são `^["USER"]Shared("msg")` e `^|"USER"|Shared("msg")`.

**c) Como testar:** O passo 2 deve mostrar que a global não existe em `LABSTUDY`; o passo 3 deve mostrar o conteúdo.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

```
LABSTUDY>ZN "USER"

USER>KILL ^Shared
USER>SET ^Shared("msg") = "escrito em USER"

USER>ZN "LABSTUDY"

LABSTUDY>WRITE $DATA(^Shared("msg")), !
0

LABSTUDY>WRITE "[", $GET(^Shared("msg")), "]", !
[]

LABSTUDY>WRITE ^["USER"]Shared("msg"), !
escrito em USER

LABSTUDY>WRITE ^|"USER"|Shared("msg"), !
escrito em USER

LABSTUDY>SET ns = "USER"
LABSTUDY>WRITE ^[ns]Shared("msg"), !
escrito em USER

LABSTUDY>SET ^["USER"]Shared("resposta") = "escrito de LABSTUDY"

LABSTUDY>ZN "USER"

USER>ZWRITE ^Shared
^Shared("msg")="escrito em USER"
^Shared("resposta")="escrito de LABSTUDY"

USER>ZN "LABSTUDY"
```

**Por que cada decisão:**

- **O passo 2 devolveu `0` e vazio, sem erro.** Isso é importante: não é que a leitura tenha falhado — é que, do ponto de vista de `LABSTUDY`, aquela global simplesmente **não existe**. São duas árvores diferentes, em armários diferentes, que por acaso têm o mesmo nome. O Capítulo 0 disse isso com a analogia das salas; agora você viu acontecer.
- **As três formas de referência estendida produzem o mesmo resultado.** A forma com variável (`^[ns]Shared`) é a mais útil na prática, porque permite escrever um utilitário que varre vários namespaces.
- **A escrita funcionou nos dois sentidos.** Referência estendida não é só leitura.
- **E aqui está o ponto que fecha o capítulo:** se, em vez de usar referência estendida, um administrador tivesse criado um **mapeamento** de `^Shared` de `LABSTUDY` para o banco de `USER`, então o passo 2 teria funcionado normalmente, sem sintaxe especial nenhuma. É exatamente isso que mapeamento faz: torna a global de outro armário visível como se fosse local. A referência estendida resolve caso a caso, no código; o mapeamento resolve de vez, na configuração.

---

### Exercício 8.5 — PROJETO CONTÍNUO: cache, área de preparação e inspetor

**a) Enunciado:** Evolua o sistema:

1. Crie `LabStudy.Cache`, que usa **PPG** para guardar resultados caros dentro da sessão:
   - `ClassMethod Put(key, value)`
   - `ClassMethod Get(key, Output found)`
   - `ClassMethod Clear()`
   - `ClassMethod Stats(Output entries, Output hits, Output misses)` — contadores mantidos na própria PPG.
2. Crie `LabStudy.Staging`, que usa **global temporária compartilhada** como área de preparação de importação:
   - `ClassMethod Reset(batchId)`
   - `ClassMethod AddRow(batchId, rawLine)` — grava a linha crua numerada
   - `ClassMethod RowCount(batchId)`
   - `ClassMethod Promote(batchId) As %Integer` — lê as linhas cruas no formato `codigo;valor;unidade`, cria os exames de verdade para um paciente, **dentro de uma transação**, e devolve quantos criaram.
3. Crie `LabStudy.StorageInfo` com:
   - `ClassMethod ClassGlobals(className)` — imprime as globais de dados, índice e stream daquela classe, com a contagem de nós de primeiro nível de cada uma;
   - `ClassMethod NextId(className)` — mostra o valor atual do contador de ID.
4. Em `LabStudy.Patient`, acrescente `ClassMethod ShowCached(id)` que usa o cache: na primeira chamada monta o texto e guarda; nas seguintes, devolve do cache.
5. Suba `LabStudy.App` para `"0.9"` e acrescente `ClassMethod StorageReport()`.

**b) Dica:** Para o `Promote`, `$PIECE(linha, ";", n)` separa os campos. Use a transação do Capítulo 5 e o padrão `entryLevel`.

**c) Como testar:**

```
LABSTUDY>DO ##class(LabStudy.Cache).Clear()
LABSTUDY>DO ##class(LabStudy.Patient).ShowCached(2)
LABSTUDY>DO ##class(LabStudy.Patient).ShowCached(2)
LABSTUDY>DO ##class(LabStudy.App).StorageReport()
```

A segunda chamada deve indicar que veio do cache.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

`src/LabStudy/Cache.cls`:

```objectscript
/// Session cache built on a process private global.
/// Everything here dies with the process, on purpose.
Class LabStudy.Cache Extends %RegisteredObject
{

/// Stores a value under a key.
ClassMethod Put(key As %String, value As %String) As %Status
{
    if $DATA(^||LabStudyCache("data", key)) = 0 {
        set ^||LabStudyCache("count") = $GET(^||LabStudyCache("count"), 0) + 1
    }

    set ^||LabStudyCache("data", key) = value
    quit $$$OK
}

/// Reads a value. The found flag says whether the key existed.
ClassMethod Get(key As %String, Output found As %Boolean) As %String
{
    set found = 0

    if $DATA(^||LabStudyCache("data", key)) {
        set found = 1
        set ^||LabStudyCache("hits") = $GET(^||LabStudyCache("hits"), 0) + 1
        quit ^||LabStudyCache("data", key)
    }

    set ^||LabStudyCache("misses") = $GET(^||LabStudyCache("misses"), 0) + 1
    quit ""
}

/// Empties the whole cache, counters included.
ClassMethod Clear() As %Status
{
    kill ^||LabStudyCache
    quit $$$OK
}

/// Reports the cache counters.
ClassMethod Stats(Output entries As %Integer, Output hits As %Integer, Output misses As %Integer) As %Status
{
    set entries = $GET(^||LabStudyCache("count"), 0)
    set hits    = $GET(^||LabStudyCache("hits"), 0)
    set misses  = $GET(^||LabStudyCache("misses"), 0)
    quit $$$OK
}

}
```

`src/LabStudy/Staging.cls`:

```objectscript
/// Import staging area, built on a shared temporary global.
/// Content is disposable: it is not journaled and does not survive a restart.
Class LabStudy.Staging Extends %RegisteredObject
{

/// Clears one batch.
ClassMethod Reset(batchId As %String) As %Status
{
    kill ^IRIS.TempLabStudy("staging", batchId)
    quit $$$OK
}

/// Appends one raw line to a batch.
ClassMethod AddRow(batchId As %String, rawLine As %String) As %Integer
{
    set n = $INCREMENT(^IRIS.TempLabStudy("staging", batchId))
    set ^IRIS.TempLabStudy("staging", batchId, n) = rawLine
    quit n
}

/// How many raw lines the batch holds.
ClassMethod RowCount(batchId As %String) As %Integer
{
    quit $GET(^IRIS.TempLabStudy("staging", batchId), 0)
}

/// Turns raw lines into real exams, all or nothing.
/// Raw format: testCode;value;unit
ClassMethod Promote(batchId As %String, patientId As %String) As %Integer
{
    set patient = ##class(LabStudy.Patient).%OpenId(patientId)
    if '$ISOBJECT(patient) {
        write "Patient not found: ", patientId, !
        quit 0
    }

    set entryLevel = $TLEVEL
    tstart

    set created = 0
    set n = ""

    for {
        set n = $ORDER(^IRIS.TempLabStudy("staging", batchId, n))
        quit:n=""

        set raw = ^IRIS.TempLabStudy("staging", batchId, n)
        set code = $PIECE(raw, ";", 1)

        if code = "" {
            write "Line ", n, " has no test code. Cancelling batch.", !
            while $TLEVEL > entryLevel { trollback 1 }
            quit
        }

        set exam = ##class(LabStudy.Exam).%New(code)
        set exam.ResultValue = $PIECE(raw, ";", 2)
        set exam.Unit = $PIECE(raw, ";", 3)
        set exam.Patient = patient

        set sc = exam.%Save()
        if $$$ISERR(sc) {
            write "Line ", n, " failed:", !
            do $SYSTEM.Status.DisplayError(sc)
            while $TLEVEL > entryLevel { trollback 1 }
            quit
        }

        set created = created + 1
    }

    if $TLEVEL > entryLevel {
        tcommit
        do ..Reset(batchId)
        quit created
    }

    quit 0
}

}
```

`src/LabStudy/StorageInfo.cls`:

```objectscript
/// Inspects where a persistent class actually stores its data.
Class LabStudy.StorageInfo Extends %RegisteredObject
{

/// Counts first level nodes of a global given by name.
ClassMethod CountNodes(globalName As %String) As %Integer [ Private ]
{
    set total = 0
    set key = ""
    for {
        set key = $ORDER(@globalName@(key))
        quit:key=""
        set total = total + 1
    }
    quit total
}

/// Prints the storage globals of a class and how many nodes each holds.
ClassMethod ClassGlobals(className As %String) As %Status
{
    set dataGlobal   = "^"_className_"D"
    set indexGlobal  = "^"_className_"I"
    set streamGlobal = "^"_className_"S"

    write "Storage of ", className, !
    write "  data   ", dataGlobal, "  nodes: ", ..CountNodes(dataGlobal), !
    write "  index  ", indexGlobal, "  nodes: ", ..CountNodes(indexGlobal), !
    write "  stream ", streamGlobal, "  nodes: ", ..CountNodes(streamGlobal), !
    write "  next id counter: ", $GET(@dataGlobal, 0), !
    quit $$$OK
}

/// Current value of the id counter of a class.
ClassMethod NextId(className As %String) As %Integer
{
    set dataGlobal = "^"_className_"D"
    quit $GET(@dataGlobal, 0)
}

}
```

Acrescente a `src/LabStudy/Patient.cls`:

```objectscript
/// Shows a patient, using the session cache to avoid rebuilding the text.
ClassMethod ShowCached(id As %String) As %Status
{
    set key = "patient:"_id
    set text = ##class(LabStudy.Cache).Get(key, .found)

    if found {
        write "(from cache) ", text, !
        quit $$$OK
    }

    set patient = ..%OpenId(id)
    if '$ISOBJECT(patient) {
        write "Patient not found: ", id, !
        quit $$$OK
    }

    set text = patient.RecordNumber_" | "_patient.Name_" | age "_patient.Age
               _" | exams "_patient.Exams.Count()

    do ##class(LabStudy.Cache).Put(key, text)
    write "(computed)   ", text, !
    quit $$$OK
}
```

E em `src/LabStudy/App.cls`:

```objectscript
Parameter VERSION = "0.9";

/// Reports where the system stores its data and how the cache is doing.
ClassMethod StorageReport() As %Status
{
    do ##class(LabStudy.StorageInfo).ClassGlobals("LabStudy.Patient")
    write !
    do ##class(LabStudy.StorageInfo).ClassGlobals("LabStudy.Exam")
    write !

    do ##class(LabStudy.Cache).Stats(.entries, .hits, .misses)
    write "Session cache: ", entries, " entries, ", hits, " hits, ", misses, " misses", !
    quit $$$OK
}
```

Execução esperada:

```
LABSTUDY>DO ##class(LabStudy.Cache).Clear()

LABSTUDY>DO ##class(LabStudy.Patient).ShowCached(2)
(computed)   REG-000002 | Bruno Lima | age 47 | exams 1

LABSTUDY>DO ##class(LabStudy.Patient).ShowCached(2)
(from cache) REG-000002 | Bruno Lima | age 47 | exams 1

LABSTUDY>DO ##class(LabStudy.Staging).Reset("batch1")
LABSTUDY>WRITE ##class(LabStudy.Staging).AddRow("batch1","CHOL;190;mg/dL"), !
1
LABSTUDY>WRITE ##class(LabStudy.Staging).AddRow("batch1","TRIG;150;mg/dL"), !
2
LABSTUDY>WRITE ##class(LabStudy.Staging).RowCount("batch1"), !
2

LABSTUDY>WRITE ##class(LabStudy.Staging).Promote("batch1", 2), " exams promoted", !
2 exams promoted

LABSTUDY>WRITE ##class(LabStudy.Staging).RowCount("batch1"), !
0

LABSTUDY>DO ##class(LabStudy.Staging).Reset("batch2")
LABSTUDY>DO ##class(LabStudy.Staging).AddRow("batch2","UREA;40;mg/dL")
LABSTUDY>DO ##class(LabStudy.Staging).AddRow("batch2",";99;mg/dL")
LABSTUDY>WRITE ##class(LabStudy.Staging).Promote("batch2", 2), " exams promoted", !
Line 2 has no test code. Cancelling batch.
0 exams promoted

LABSTUDY>DO ##class(LabStudy.App).StorageReport()
Storage of LabStudy.Patient
  data   ^LabStudy.PatientD  nodes: 1
  index  ^LabStudy.PatientI  nodes: 3
  stream ^LabStudy.PatientS  nodes: 0
  next id counter: 2

Storage of LabStudy.Exam
  data   ^LabStudy.ExamD  nodes: 3
  index  ^LabStudy.ExamI  nodes: 2
  stream ^LabStudy.ExamS  nodes: 1
  next id counter: 4

Session cache: 1 entries, 1 hits, 1 misses
```

**Por que cada decisão:**

- **O cache é PPG, não global persistente.** Um cache de sessão que sobrevivesse à sessão seria pior que inútil: entregaria dados velhos para o próximo usuário. A escolha do meio **é** a decisão de projeto aqui, e ela codifica a semântica desejada — "isto vale só para mim, só agora".
- **Os contadores do cache moram dentro da própria PPG.** Se estivessem numa global persistente, seriam compartilhados entre processos e não diriam nada útil sobre a sessão. Coerência de meio.
- **`Get` devolve o valor pelo `QUIT` e o indicador de acerto por `Output`.** Sem esse indicador, um valor vazio legitimamente guardado seria indistinguível de "não encontrei". É o padrão do Capítulo 3 resolvendo um problema real.
- **A área de preparação é global temporária compartilhada, não PPG.** A diferença é intencional: uma importação pode ser alimentada por um processo (que recebe o arquivo) e promovida por outro (que processa o lote). PPG impediria isso. Mas os dados crus também não merecem journaling nem backup — eles são descartáveis assim que virarem exames de verdade. Global temporária é exatamente o meio-termo certo.
- **`Promote` é transacional e "tudo ou nada".** O segundo lote falhou inteiro por causa de uma linha, e **nenhum** exame foi criado — nem mesmo o da linha 1, que era válida. Compare com o `ImportBatch` do Capítulo 4, que usava `continue` e importava o que dava. As duas políticas são defensáveis; o que não é defensável é não ter escolhido conscientemente.
- **`Promote` limpa a área de preparação só depois do commit.** Se limpasse antes, uma falha deixaria você sem os dados originais para reprocessar.
- **Note que o lote cancelado deixou os dados crus intactos** — `^IRIS.TempLabStudy` não é journalizada e portanto **não foi desfeita pelo rollback**. Isso é uma consequência direta do que você aprendeu, e neste caso é exatamente o comportamento desejado: você quer poder corrigir a linha 2 e tentar de novo. Se a área de preparação fosse uma global persistente dentro da transação, o rollback teria apagado o lote junto, e você perderia os dados a corrigir. **A escolha do meio decidiu o comportamento em caso de falha** — este é o aprendizado mais valioso do exercício.
- **`StorageInfo` monta o nome da global a partir do nome da classe** e usa indireção para percorrê-la. Isso funciona porque a convenção de nomes do armazenamento padrão é previsível — mas **só** para armazenamento padrão. Uma classe com `Storage` customizado quebraria essa suposição. O jeito rigoroso seria ler a definição de armazenamento pelo dicionário de classes; ficou como está por simplicidade, e é honesto reconhecer a limitação.
- **`^LabStudy.ExamS` com 1 nó** é o stream do laudo que gravamos no Capítulo 4, ainda lá. **`^LabStudy.PatientS` com 0** confirma que `Patient` não tem stream algum.

---

## 8. Quiz do capítulo

**Q1.** Qual é a sintaxe de uma global privada do processo?

- A) `^Nome`
- B) `^||Nome`
- C) `%Nome`
- D) `^IRIS.TempNome`

---

**Q2.** Qual afirmação sobre globais privadas do processo está correta?

- A) São compartilhadas entre processos e journalizadas.
- B) São privadas do processo, apagadas quando ele termina, e não são journalizadas.
- C) Sobrevivem ao reinício da instância.
- D) Comportam-se como variáveis locais e morrem ao fim do método.

---

**Q3.** `$DATA(^X("a"))` devolveu `10`. O que isso significa?

- A) O nó tem valor e não tem descendentes.
- B) O nó não existe.
- C) O nó não tem valor próprio, mas tem descendentes.
- D) O nó tem valor e tem descendentes.

---

**Q4.** Qual é a diferença entre `$ORDER` e `$QUERY`?

- A) Nenhuma.
- B) `$ORDER` percorre um nível; `$QUERY` percorre toda a árvore, devolvendo a referência completa do próximo nó.
- C) `$ORDER` percorre a árvore toda; `$QUERY` percorre um nível.
- D) `$QUERY` só funciona em variáveis locais.

---

**Q5.** O que `KILL ^Dados("a")` faz?

- A) Apaga apenas o valor do nó `^Dados("a")`.
- B) Apaga o nó `^Dados("a")` e todos os seus descendentes.
- C) Apaga a global `^Dados` inteira.
- D) Não faz nada se o nó tiver filhos.

---

**Q6.** Como ler uma global que pode não existir, sem risco de erro?

- A) `WRITE ^X("y")`
- B) `WRITE $GET(^X("y"))`
- C) `WRITE $ORDER(^X("y"))`
- D) `WRITE $QUERY(^X("y"))`

---

**Q7.** Numa classe `LabStudy.Patient` com armazenamento padrão, onde ficam os dados dos objetos?

- A) `^LabStudy.Patient`
- B) `^LabStudy.PatientD`
- C) `^LabStudy.PatientI`
- D) `^PATIENT`

---

**Q8.** O que fica guardado na **raiz** da global de dados de uma classe persistente?

- A) O primeiro objeto.
- B) O contador do último ID atribuído.
- C) O nome da tabela SQL.
- D) A definição de armazenamento.

---

**Q9.** Você declarou `SqlTableName = PATIENT` na classe `LabStudy.Patient`. Qual é o nome da global de dados?

- A) `^PATIENT`
- B) `^LabStudy.PATIENTD`
- C) `^LabStudy.PatientD` — `SqlTableName` afeta só o SQL.
- D) Depende do namespace.

---

**Q10.** Qual afirmação sobre índices no armazenamento está correta?

- A) Índices e dados ficam na mesma global.
- B) Um índice comum guarda o ID como último subscrito e o nó fica vazio; um índice único guarda o ID no valor do nó.
- C) Índices são gravados apenas na memória.
- D) Índices únicos não são armazenados.

---

**Q11.** Como acessar a global `^Config` do namespace `USER` estando em `LABSTUDY`?

- A) `^Config` — globais são compartilhadas entre namespaces.
- B) `^["USER"]Config` ou `^|"USER"|Config`
- C) `ZN "USER"` é a única forma.
- D) `^USER.Config`

---

**Q12.** O que `MERGE ^A = ^B` faz?

- A) Apaga `^A` e copia `^B` inteira.
- B) Copia `^B` e toda a sua subárvore para `^A`, sem apagar o que já havia em `^A`.
- C) Move `^B` para `^A`.
- D) Compara as duas globais.

---

**Q13.** Uma global temporária compartilhada (`^IRIS.Temp...`) sobrevive a quê?

- A) Ao fim do processo, mas não ao reinício da instância.
- B) Ao reinício da instância.
- C) Nem ao fim do processo.
- D) A tudo, como qualquer global.

---

**Q14.** Você precisa acumular 50.000 linhas durante um processamento, sem compartilhar com outros processos e sem que os dados precisem sobreviver à sessão. Qual meio é o mais adequado?

- A) Variável local.
- B) Global privada do processo (`^||`).
- C) Global persistente.
- D) Uma classe persistente.

---

**Q15.** Por que uma global mapeada para um banco sem journaling não é desfeita por rollback?

- A) Porque o rollback não funciona em globais.
- B) Porque o rollback usa o journal para restaurar valores anteriores, e sem journal não há o que restaurar.
- C) Porque globais sem journal são somente leitura.
- D) Porque o mapeamento desliga as transações.

---

### Gabarito comentado

**Q1 — Resposta: B.**
- **B está certa:** `^||Nome` — circunflexo seguido de dois pipes.
- **A está errada:** essa é uma global persistente comum.
- **C está errada:** o `%` marca itens do sistema.
- **D está errada:** esse é o padrão de nome de global temporária compartilhada.

**Q2 — Resposta: B.**
- **B está certa:** privada, com o tempo de vida do processo, sem journaling.
- **A está errada:** ninguém mais a enxerga, e ela não é journalizada.
- **C está errada:** ela nem sobrevive ao fim do processo.
- **D está errada:** ela sobrevive ao fim do método, diferentemente de uma variável local.

**Q3 — Resposta: C.**
- **C está certa:** dezenas indicam descendentes, unidades indicam valor. `10` = tem filhos, não tem valor.
- **A está errada:** isso seria `1`.
- **B está errada:** isso seria `0`.
- **D está errada:** isso seria `11`.

**Q4 — Resposta: B.**
- **B está certa:** `$ORDER` anda num nível e devolve o subscrito; `$QUERY` percorre toda a árvore e devolve a referência completa.
- **A está errada:** têm finalidades distintas.
- **C está errada:** inverte os dois.
- **D está errada:** `$QUERY` funciona em globais e em variáveis locais.

**Q5 — Resposta: B.**
- **B está certa:** `KILL` remove o nó **e toda a subárvore** abaixo dele.
- **A está errada:** isso seria `ZKILL`.
- **C está errada:** apagar a global inteira seria `KILL ^Dados`, sem subscrito.
- **D está errada:** ele apaga justamente também os filhos.

**Q6 — Resposta: B.**
- **B está certa:** `$GET` devolve vazio (ou um padrão) em vez de causar `<UNDEFINED>`.
- **A está errada:** leitura direta de nó inexistente gera erro.
- **C está errada:** `$ORDER` devolve o próximo subscrito, não o valor.
- **D está errada:** `$QUERY` devolve uma referência, não o valor do nó pedido.

**Q7 — Resposta: B.**
- **B está certa:** a convenção do armazenamento padrão é `^Pacote.ClasseD` para dados.
- **A está errada:** essa global não é usada.
- **C está errada:** essa é a de índices.
- **D está errada:** o nome da tabela SQL não determina o nome da global.

**Q8 — Resposta: B.**
- **B está certa:** a raiz guarda o contador do último ID, e é por isso que IDs não são reaproveitados após exclusões.
- **A está errada:** o primeiro objeto fica em `(1)`.
- **C está errada:** o nome da tabela vive no dicionário de classes.
- **D está errada:** a definição de armazenamento fica na definição da classe.

**Q9 — Resposta: C.**
- **C está certa:** `SqlTableName` afeta apenas a projeção SQL; a global vem do nome da classe.
- **A e B estão erradas:** confundem os dois mundos.
- **D está errada:** o nome não muda com o namespace, embora a global fisicamente resida no banco daquele namespace.

**Q10 — Resposta: B.**
- **B está certa:** é exatamente o que se observa ao inspecionar `^Pacote.ClasseI`. Onde só cabe um ID, ele vai no valor; onde cabem vários, vira subscrito.
- **A está errada:** dados e índices ficam em globais separadas no armazenamento padrão.
- **C está errada:** índices são persistidos.
- **D está errada:** índices únicos são armazenados, apenas com estrutura diferente.

**Q11 — Resposta: B.**
- **B está certa:** referência estendida, nas duas formas equivalentes.
- **A está errada:** cada namespace tem a sua própria árvore, salvo mapeamento explícito.
- **C está errada:** trocar de namespace funciona, mas não é a única forma.
- **D está errada:** essa sintaxe não existe.

**Q12 — Resposta: B.**
- **B está certa:** `MERGE` copia a subárvore e mescla com o que já existe no destino.
- **A está errada:** ele não apaga o destino antes.
- **C está errada:** a origem permanece intacta.
- **D está errada:** ele não compara nada.

**Q13 — Resposta: A.**
- **A está certa:** sobrevive ao fim do processo, mas é apagada quando a instância reinicia.
- **B está errada:** é exatamente o que ela não sobrevive.
- **C está errada:** ela sobrevive ao processo, diferentemente de local e PPG.
- **D está errada:** ela é temporária por definição.

**Q14 — Resposta: B.**
- **B está certa:** volume alto, privado do processo, descartável ao fim da sessão — a descrição exata da PPG.
- **A está errada:** 50.000 linhas em memória de processo é desperdício e pode pressionar o espaço disponível.
- **C está errada:** persistir e journalizar o que é descartável é custo puro.
- **D está errada:** classe persistente traz validação e índices que aqui não são necessários e custam caro.

**Q15 — Resposta: B.**
- **B está certa:** o rollback restaura valores a partir do journal; sem journal, não há registro do valor anterior.
- **A está errada:** rollback funciona em globais journalizadas.
- **C está errada:** elas continuam graváveis.
- **D está errada:** o mapeamento não desliga transações; ele apenas aponta para um banco cujo journaling está desativado.

---

## 9. Resumo relâmpago

1. Quatro meios: **variável local**, **PPG (`^||`)**, **global temporária compartilhada** e **global persistente (`^`)**. Escolha o **menos duradouro que resolva**.
2. Local: memória do processo, morre com o bloco. PPG: privada do processo, morre com o processo. Temporária: compartilhada, morre no reinício da instância. Persistente: compartilhada, permanente, journalizada.
3. **PPG e temporária não são journalizadas** — não participam de transações nem são desfeitas por rollback.
4. Uma global é uma **árvore**: nós, subscritos, níveis. Um nó pode ter **valor e filhos ao mesmo tempo**.
5. Subscritos não são declarados e vêm sempre **ordenados**: vazio, depois números, depois texto.
6. **`$DATA`**: `0` inexistente, `1` só valor, `10` só filhos, `11` valor e filhos. Dezenas = filhos, unidades = valor.
7. **`$GET(ref)`** e **`$GET(ref, padrão)`** leem sem risco de `<UNDEFINED>`. Use sempre em produção.
8. **`$ORDER(ref)`** anda **num nível**; `-1` como segundo argumento anda ao contrário; o terceiro argumento devolve o valor.
9. **`$QUERY(@ref)`** percorre **toda a árvore** e devolve a referência completa do próximo nó.
10. **`KILL`** apaga o nó **e toda a subárvore**. **`ZKILL`** apaga só o valor, preservando os filhos.
11. **`MERGE destino = origem`** copia uma subárvore inteira, sem limpar o destino antes.
12. Armazenamento padrão de classe: **`^Pacote.ClasseD`** (dados), **`^Pacote.ClasseI`** (índices), **`^Pacote.ClasseS`** (streams).
13. A **raiz da global de dados guarda o contador de ID** — por isso IDs não são reaproveitados.
14. Os dados de um objeto ficam num nó como **`$LISTBUILD`**, na ordem definida no bloco `Storage`; a posição 1 é reservada ao nome da classe.
15. **Índice comum**: ID como último subscrito, nó vazio. **Índice único**: ID no valor do nó.
16. `SqlTableName` afeta o **SQL**; o nome da **global** vem do nome da **classe**.
17. **Referência estendida**: `^["USER"]Config` ou `^|"USER"|Config`, e o namespace pode vir de variável.
18. **Mapeamento** (globais, rotinas, pacotes) faz o conteúdo de outro banco aparecer como se fosse local, sem sintaxe especial.
19. **Journaling é configurado por banco**, não por global nem por namespace.
20. Para dados de negócio, prefira **classe persistente** à global crua: você ganha validação, índices, SQL e evolução de esquema.

---

## 10. Cartões de memorização

**Frente:** Quais são os quatro meios de armazenamento do IRIS?
**Verso:** Variável local, PPG (`^||`), global temporária compartilhada e global persistente (`^`).

**Frente:** Qual a sintaxe de uma global privada do processo?
**Verso:** `^||Nome` — circunflexo seguido de dois pipes.

**Frente:** Quanto tempo vive uma PPG? Quem a enxerga?
**Verso:** Até o fim do processo. Só o próprio processo. Não é journalizada.

**Frente:** Uma global temporária compartilhada sobrevive a quê?
**Verso:** Ao fim do processo, mas não ao reinício da instância. Não é journalizada.

**Frente:** O que significam os quatro valores de `$DATA`?
**Verso:** `0` inexistente; `1` só valor; `10` só filhos; `11` valor e filhos. Dezenas = filhos, unidades = valor.

**Frente:** Como ler uma global sem risco de `<UNDEFINED>`?
**Verso:** `$GET(ref)` ou `$GET(ref, padrão)`.

**Frente:** Qual a diferença entre `$ORDER` e `$QUERY`?
**Verso:** `$ORDER` anda num nível e devolve o subscrito. `$QUERY` percorre toda a árvore e devolve a referência completa.

**Frente:** Como percorrer subscritos ao contrário?
**Verso:** `$ORDER(^X(k), -1)`.

**Frente:** Qual é o padrão do laço com `$ORDER`?
**Verso:** `set k="" for { set k=$ORDER(^X(k)) quit:k="" ... }`.

**Frente:** O que `KILL ^X("a")` remove?
**Verso:** O nó e **toda a subárvore** abaixo dele. Para apagar só o valor, `ZKILL`.

**Frente:** O que `MERGE ^A = ^B` faz?
**Verso:** Copia `^B` e toda a sua subárvore para `^A`, sem limpar o destino antes.

**Frente:** Em que globais uma classe persistente guarda dados, índices e streams?
**Verso:** `^Pacote.ClasseD`, `^Pacote.ClasseI` e `^Pacote.ClasseS`.

**Frente:** O que fica na raiz de `^Pacote.ClasseD`?
**Verso:** O contador do último ID atribuído. Por isso IDs não são reaproveitados.

**Frente:** Como um objeto é gravado dentro do nó de dados?
**Verso:** Como uma lista `$LISTBUILD`, na ordem do bloco `Storage`; a posição 1 é reservada ao nome da classe.

**Frente:** Como um índice comum e um índice único diferem no armazenamento?
**Verso:** Comum: `("Idx", valor, id) = ""`. Único: `("Idx", valor) = id`.

**Frente:** `SqlTableName = PATIENT` muda o nome da global?
**Verso:** Não. `SqlTableName` afeta só o SQL; a global vem do nome da classe.

**Frente:** Como acessar uma global de outro namespace?
**Verso:** Referência estendida: `^["USER"]Config` ou `^|"USER"|Config`. O namespace pode vir de variável.

**Frente:** O que é mapeamento de globais?
**Verso:** Configuração que faz uma global de outro banco aparecer como local naquele namespace, sem sintaxe especial.

**Frente:** Journaling é configurado por quê?
**Verso:** Por **banco de dados** — não por global nem por namespace.

**Frente:** Preciso acumular muitas linhas, só para mim, só nesta sessão. Qual meio?
**Verso:** PPG (`^||`).

**Frente:** Preciso de uma área de trabalho compartilhada e descartável. Qual meio?
**Verso:** Global temporária compartilhada (`^IRIS.Temp...`).

**Frente:** Por que uma global sem journaling não volta atrás num rollback?
**Verso:** Porque o rollback restaura valores a partir do journal. Sem journal, não há registro do valor anterior.

---

Digite CONTINUAR para o próximo capítulo.
