# Apostila InterSystems ObjectScript Specialist
## Capítulo 1 — T1.1 Uses Classes (Usando classes)

> Este é o primeiro tópico do domínio **T1 — Manages Data Model**, o segundo de maior peso na prova (23 questões de 76). Vamos com calma e com profundidade.

---

## 1. Objetivo do capítulo

Ao terminar este capítulo, você será capaz de:

1. Explicar o que é uma **classe**, o que é um **objeto** e o que é uma **referência de objeto (OREF)**.
2. Diferenciar os três tipos fundamentais de classe do IRIS: **`%RegisteredObject`**, **`%Persistent`** e **`%SerialObject`** — e saber quando usar cada um.
3. Escrever a definição de uma classe do zero, entendendo cada palavra e cada símbolo.
4. Criar um objeto na memória com **`%New()`**, gravá-lo em disco com **`%Save()`**, recuperá-lo com **`%OpenId()`** e apagá-lo com **`%DeleteId()`**.
5. Entender o que é o **ID** de um objeto persistente e de onde ele vem.
6. Verificar se uma operação deu certo, usando **`%Status`**, **`$$$ISOK`**, **`$$$ISERR`** e `$SYSTEM.Status.DisplayError()`.
7. Usar **herança**: herança simples, herança múltipla, superclasse primária e a palavra-chave `Inheritance`.
8. Reconhecer as palavras-chave de classe que a prova cobra: `Abstract`, `Final`, `Extends`, `SqlTableName`.
9. Compilar, listar e apagar classes pelo Terminal.
10. Criar a primeira classe persistente do projeto contínuo: **`LabStudy.Patient`**.

---

## 2. O conceito em linguagem de gente

### 2.1 Retomando: molde e peça

No Capítulo 0 usamos a forma de bolo. Vamos trocar a analogia por uma mais próxima do nosso projeto de laboratório, porque ela vai nos servir o capítulo inteiro.

Imagine a **ficha de papel** que a recepcionista do laboratório preenche quando um paciente chega.

Existe **um formulário em branco**, impresso na gráfica, que define o que toda ficha tem: um campo "Nome", um campo "Data de nascimento", um campo "Número do registro". Esse formulário em branco é a **classe**.

Cada ficha preenchida — a do João, a da Maria — é um **objeto**.

O formulário em branco é único. As fichas preenchidas são muitas. E, mais importante: **o formulário em branco não guarda dados de ninguém**. Ele só diz qual é o formato.

### 2.2 A grande pergunta: a ficha vai para o armário ou vai para o lixo?

Aqui está a decisão mais importante deste capítulo, e é uma decisão que a prova cobra o tempo todo.

Quando a recepcionista preenche uma ficha, ela pode fazer três coisas diferentes com o papel:

**Situação 1 — a ficha é um rascunho.**
Ela pega um papel qualquer, anota os dados para conferir alguma coisa, usa e joga fora no fim do expediente. Nada foi arquivado. Amanhã aquele papel não existe mais.

No IRIS, isso é uma classe que herda de **`%RegisteredObject`**. Ela existe **só na memória**, enquanto o programa roda. Não tem armário, não tem disco, não sobrevive. É perfeita para cálculos, para agrupar informação temporária, para organizar código.

**Situação 2 — a ficha vai para o armário.**
Ela preenche a ficha, carimba um número de registro nela e guarda no arquivo de aço. Amanhã, ano que vem, a ficha continua lá. Você pode ir ao armário, procurar pelo número e trazer a ficha de volta.

No IRIS, isso é uma classe que herda de **`%Persistent`**. Ela é **gravada em disco**, ganha um **número de identificação (ID)** e pode ser recuperada depois. É a classe que representa dados de verdade: pacientes, exames, resultados.

**Situação 3 — a ficha é uma etiqueta colada dentro de outra ficha.**
Pense no endereço do paciente. O endereço tem rua, número, cidade, CEP. É um conjunto de campos que anda junto. Mas o endereço **não tem vida própria**: ninguém arquiva um endereço sozinho no armário. Ele só existe grudado na ficha de alguém.

No IRIS, isso é uma classe que herda de **`%SerialObject`**. Ela agrupa campos, mas **não pode ser guardada sozinha**. Ela é sempre gravada **dentro** de um objeto persistente que a contém. Isso se chama **objeto embutido** (*embedded object*).

Guarde esta tabelinha mental, porque ela vale muitos pontos na prova:

| Se você precisa de... | Herde de |
|---|---|
| um objeto temporário, só na memória | `%RegisteredObject` |
| um objeto gravado em disco, com ID próprio, recuperável | `%Persistent` |
| um pedaço de dados que só existe dentro de outro objeto | `%SerialObject` |

E uma relação de família importante: **`%RegisteredObject` é a base de todas**. `%Persistent` e `%SerialObject` descendem dela por caminhos internos do IRIS. Ou seja: tudo que um `%RegisteredObject` sabe fazer, um `%Persistent` também sabe. O contrário não é verdade — um `%RegisteredObject` **não** sabe gravar em disco.

### 2.3 O que é uma OREF

Agora uma ideia nova e essencial.

Volte ao balcão do laboratório. Você pede a ficha do paciente 42. O atendente vai ao armário, pega a ficha e coloca na sua mão. A partir daí, você não fica gritando "paciente 42!" toda vez: você simplesmente olha o papel que está na sua mão.

Esse "papel na sua mão" é a **OREF** — abreviação de *Object Reference*, referência de objeto.

Uma OREF **não é o objeto**. É um endereço, um bilhete, que aponta para o objeto que está na memória. Quando você faz:

```
SET patient = ##class(LabStudy.Patient).%New()
```

a variável `patient` não contém um paciente. Ela contém uma **OREF**: um bilhete que diz "o paciente está na posição tal da memória".

E tem uma coisa curiosa e útil: se você mandar escrever uma OREF na tela, o IRIS mostra algo assim:

```
LABSTUDY>WRITE patient
1@LabStudy.Patient
```

Leia isso como: *"o bilhete número 1, apontando para um objeto da classe LabStudy.Patient"*. Aquele número antes do arroba é apenas um contador interno de bilhetes daquela sessão. **Ele não é o ID do objeto no banco.** Confundir os dois é um clássico de prova, e vamos separar bem os dois na seção 5.

### 2.4 O que é o ID

Quando você guarda uma ficha no armário, ela recebe um número de registro carimbado. Esse número é único naquele armário e serve para você achar a ficha depois.

No IRIS, esse número é o **ID** do objeto persistente.

Fatos sobre o ID que você precisa saber:

- Ele **só existe depois que o objeto foi gravado** com sucesso. Antes de gravar, o objeto ainda não tem ID.
- Por padrão, o IRIS atribui um **número inteiro positivo crescente**: 1, 2, 3, 4...
- Esse número vem de um contador que o IRIS mantém sozinho, em disco.
- **IDs não são reaproveitados.** Se você apagar o objeto 3, o próximo objeto criado não vira 3; ele continua a sequência.
- Só classes **persistentes** têm ID. Um `%RegisteredObject` nunca terá.

### 2.5 O bônus escondido: toda classe persistente vira uma tabela SQL

Este ponto costuma surpreender quem está começando, e é muito cobrado.

Quando você define uma classe persistente `LabStudy.Patient` e compila, o IRIS **automaticamente** cria uma tabela SQL correspondente. Sem você pedir. Sem escrever `CREATE TABLE`.

- O **pacote** (`LabStudy`) vira o **schema** da tabela.
- O **nome da classe** (`Patient`) vira o **nome da tabela**.
- As **propriedades** viram **colunas**.
- Os **objetos gravados** viram **linhas**.

Ou seja, os mesmos dados podem ser acessados de duas maneiras: como objetos (com `%OpenId`, `patient.Name`) ou como tabela (com `SELECT Name FROM LabStudy.Patient`). É a mesma informação, no mesmo lugar em disco, com duas portas de entrada. A InterSystems chama isso de **acesso multimodelo**, e é uma das características centrais do IRIS.

---

## 3. A sintaxe explicada

### 3.1 A forma geral de uma definição de classe

```
Class Package.ClassName Extends SuperClass [ KeywordList ]
{

  ... membros da classe ...

}
```

Pedaço por pedaço:

- **`Class`** — palavra fixa. Anuncia o início da definição. **Obrigatória.**
- **`Package.ClassName`** — o nome completo. **Obrigatório.** O que vem antes do último ponto é o **pacote**; o que vem depois é o nome da classe. Pode haver vários níveis: `LabStudy.Model.Patient` tem pacote `LabStudy.Model`.
- **`Extends`** — de quem essa classe herda. **Opcional.** Se você omitir, a classe não herda de ninguém e não terá `%New()`, nem `%Save()`, nem nada — o que quase nunca é o que você quer.
- **`SuperClass`** — o nome da classe mãe. Para herdar de mais de uma, use parênteses: `Extends (ClassA, ClassB)`.
- **`[ KeywordList ]`** — as **palavras-chave de classe** (*class keywords*), entre colchetes, separadas por vírgula. **Opcional.** Exemplo: `[ Abstract ]`, `[ SqlTableName = PATIENT ]`.
- **`{ ... }`** — as chaves delimitam o corpo. **Obrigatórias.**

Um exemplo completo com tudo:

```objectscript
Class LabStudy.Patient Extends %Persistent [ SqlTableName = PATIENT ]
{
}
```

Leia em voz alta: *"a classe Patient, do pacote LabStudy, herda de %Persistent, e no SQL a tabela vai se chamar PATIENT."*

### 3.2 Regras de nomes

- O nome de um pacote ou de uma classe deve **começar com uma letra** (ou com `%`, mas `%` é reservado ao sistema — não use).
- Depois da primeira letra, pode conter **letras e números**.
- Os pontos separam os níveis do pacote. O ponto **não** é parte do nome de um nível.
- Nomes **diferenciam maiúsculas de minúsculas**. `LabStudy.Patient` e `Labstudy.Patient` são classes diferentes.
- Existem **limites de comprimento** para nomes de classe e de pacote no IRIS. Os valores exatos variam por versão: **verificar na documentação oficial** se você precisar do número preciso. Na prática, nomes razoáveis nunca esbarram nesses limites.

### 3.3 Os membros de uma classe

Dentro das chaves você pode declarar vários tipos de membro. Você não precisa usar todos; a maioria das classes usa dois ou três.

| Membro | Para que serve | Capítulo |
|---|---|---|
| `Property` | um campo do objeto (nome, idade) | 1.2 |
| `Index` | um atalho de busca sobre uma propriedade | 1.2 |
| `Parameter` | um valor fixo, decidido na compilação | este |
| `Method` | uma ação que só funciona sobre um objeto existente | 1.3 |
| `ClassMethod` | uma ação chamada direto pela classe | 1.3 |
| `Query` | uma consulta nomeada e reaproveitável | 4.6 |
| `Trigger` | código que dispara sozinho em eventos de SQL | 3.2 |
| `XData` | um bloco de dados livre dentro da classe | 4.7 |
| `Relationship` | uma ligação estruturada entre duas classes persistentes | 1.2 |

Cada membro tem sua própria sintaxe. Neste capítulo vamos usar `Property`, `Parameter` e `ClassMethod` no nível mínimo necessário — os detalhes finos vêm nos capítulos seguintes.

### 3.4 O ciclo de vida de um objeto persistente

Esta é a sequência de cinco passos que você vai repetir a vida inteira. Decore.

```
1. CRIAR    SET obj = ##class(Package.Class).%New()
2. PREENCHER SET obj.Property = valor
3. GRAVAR    SET status = obj.%Save()
4. ABRIR     SET obj2 = ##class(Package.Class).%OpenId(id)
5. APAGAR    SET status = ##class(Package.Class).%DeleteId(id)
```

Agora, um por um, com a sintaxe explicada.

---

**`%New()` — criar um objeto na memória**

```
SET patient = ##class(LabStudy.Patient).%New()
```

- `##class(LabStudy.Patient)` — aponta para a classe.
- `.%New()` — o método que **constrói** um objeto novo, vazio, na memória.
- O que ele **devolve** é uma **OREF**, não um `%Status`. Isso cai na prova.
- Se der errado (por exemplo, se a classe for abstrata), ele devolve uma **OREF nula**, que é a string vazia `""`.
- **`%New()` não grava nada em disco.** O objeto está só na memória. Não tem ID ainda.

Note o `%` no começo do nome: pelo que aprendemos no Capítulo 0, `%` marca coisas do sistema. `%New`, `%Save`, `%OpenId` são todos métodos que o IRIS te deu de presente pela herança.

---

**`%Save()` — gravar em disco**

```
SET status = patient.%Save()
```

- Repare: aqui usamos **`patient.%Save()`**, e não `##class(...)`. Por quê? Porque `%Save` é um **método de instância**: ele age **sobre um objeto específico**. Você não pode salvar "a classe"; você salva "este paciente".
- O que ele devolve é um **`%Status`**, não uma OREF. Também cai na prova.
- Se der certo, o objeto passa a existir em disco e ganha um **ID**.
- `%Save()` também **valida** o objeto antes de gravar: se uma propriedade obrigatória estiver vazia ou fora do formato, ele recusa e devolve um status de erro. As regras de validação são o assunto do Capítulo 1.2.

---

**`%Id()` — descobrir o ID depois de gravar**

```
WRITE patient.%Id()
```

Devolve o ID do objeto, como texto. Antes de um `%Save()` bem-sucedido, devolve vazio.

---

**`%OpenId()` — trazer de volta do disco**

```
SET patient = ##class(LabStudy.Patient).%OpenId(1)
```

- Aqui voltamos ao `##class(...)`, porque você ainda **não tem** o objeto: você está pedindo à classe que vá buscar.
- Devolve uma **OREF** se encontrou, ou **OREF nula (`""`)** se não encontrou.
- **Ele não dá erro** quando o ID não existe. Simplesmente devolve vazio. Se você não conferir, o erro só vai aparecer lá na frente, quando você tentar usar o objeto — e aí vem `<INVALID OREF>`.

A forma completa do método aceita mais argumentos:

```
SET patient = ##class(LabStudy.Patient).%OpenId(id, concurrency, .status)
```

- `id` — **obrigatório**. O identificador do objeto.
- `concurrency` — **opcional**. O nível de bloqueio a usar ao abrir. Tratamos disso no Capítulo 2.1, junto com transações e travas.
- `.status` — **opcional**. Repare no **ponto antes do nome**. Esse ponto significa "passe a variável **por referência**", ou seja: *"IRIS, escreva o resultado dentro desta minha variável"*. Depois da chamada, `status` conterá o motivo caso a abertura falhe. É a forma correta de descobrir **por que** não abriu.

---

**`%ExistsId()` — perguntar antes de abrir**

```
IF ##class(LabStudy.Patient).%ExistsId(1) { WRITE "existe", ! }
```

Devolve `1` se existe um objeto com aquele ID, `0` se não existe. É mais barato que abrir o objeto só para descobrir se ele existe.

---

**`%DeleteId()` — apagar do disco**

```
SET status = ##class(LabStudy.Patient).%DeleteId(1)
```

- Chamado pela **classe**, não pelo objeto, porque você identifica o que apagar pelo ID.
- Devolve um **`%Status`**.
- Existe também `%Delete(oid)`, que recebe um **OID** em vez de um ID. OID é uma forma interna e mais completa de identificar um objeto, que inclui a classe. Para o dia a dia e para a prova, `%DeleteId()` é o que você usa; saiba apenas que `%Delete()` existe e que **espera um OID, não um ID** — trocar os dois é pegadinha clássica.

---

**`%KillExtent()` — apagar tudo (cuidado)**

```
DO ##class(LabStudy.Patient).%KillExtent()
```

Apaga **todos** os objetos daquela classe, sem perguntar e sem executar as validações e regras de apagamento normais. Em estudo é ótimo para limpar a bagunça. Em produção, é perigoso. Nunca use sem ter certeza absoluta.

### 3.5 Conferindo se deu certo: o `%Status`

Vários métodos devolvem um `%Status`. Um `%Status` é um valor que carrega duas informações: **deu certo ou não**, e **se não deu, por quê**.

Você **nunca** deve ignorar um `%Status`. As três ferramentas para lidar com ele:

```
SET status = patient.%Save()

IF $$$ISOK(status) { WRITE "gravou", ! }
IF $$$ISERR(status) { DO $SYSTEM.Status.DisplayError(status) }
```

- **`$$$ISOK(status)`** — macro que devolve verdadeiro se o status for de sucesso.
- **`$$$ISERR(status)`** — macro que devolve verdadeiro se o status for de erro.
- **`$SYSTEM.Status.DisplayError(status)`** — imprime na tela a mensagem de erro legível.
- **`$SYSTEM.Status.GetErrorText(status)`** — devolve essa mensagem como texto, para você guardar em log em vez de imprimir.

Aquele `$SYSTEM` é uma variável especial do sistema (lembra do cifrão do Capítulo 0) que dá acesso a um conjunto de utilitários do IRIS. Vamos usá-la bastante.

O `%Status` completo é assunto do Capítulo 5.2. Aqui basta você adquirir o hábito: **guardou o retorno, conferiu o retorno.**

---

## 4. Exemplo comentado

Vamos criar uma classe persistente pequena e percorrer o ciclo de vida inteiro.

### 4.1 A classe

Arquivo `src/LabStudy/Sample.cls`:

```objectscript
/// A minimal persistent class used to demonstrate the object life cycle.
Class LabStudy.Sample Extends %Persistent
{

/// Code that identifies the sample, e.g. "S-0001".
Property Code As %String;

/// Type of material collected, e.g. "blood".
Property Material As %String;

}
```

Linha por linha:

- `/// A minimal persistent class...` — comentário de documentação (três barras). O IRIS guarda esse texto e exibe na documentação automática da classe.
- `Class LabStudy.Sample Extends %Persistent` — declara a classe no pacote `LabStudy`, herdando de `%Persistent`. Essa herança é o que dá a ela `%Save()`, `%OpenId()`, `%DeleteId()`, um ID automático e uma tabela SQL.
- `{` abre o corpo.
- `Property Code As %String;` — declara uma propriedade chamada `Code`, do tipo `%String` (texto). Repare no **ponto e vírgula no final**: declarações de propriedade terminam com `;`. Esquecer isso é erro de compilação.
- `Property Material As %String;` — a segunda propriedade.
- `}` fecha o corpo.

Salve com `Ctrl+S`. A extensão compila. Se compilou, o IRIS agora conhece essa classe **e** a tabela SQL `LabStudy.Sample`.

### 4.2 O ciclo de vida no Terminal

```
LABSTUDY>SET sample = ##class(LabStudy.Sample).%New()

LABSTUDY>WRITE sample
1@LabStudy.Sample

LABSTUDY>WRITE sample.%Id()

LABSTUDY>SET sample.Code = "S-0001"

LABSTUDY>SET sample.Material = "blood"

LABSTUDY>SET status = sample.%Save()

LABSTUDY>WRITE $$$ISOK(status)
1

LABSTUDY>WRITE sample.%Id()
1

LABSTUDY>SET sample = ""

LABSTUDY>SET reopened = ##class(LabStudy.Sample).%OpenId(1)

LABSTUDY>WRITE reopened.Code, " / ", reopened.Material, !
S-0001 / blood

LABSTUDY>WRITE ##class(LabStudy.Sample).%ExistsId(99)
0

LABSTUDY>SET status = ##class(LabStudy.Sample).%DeleteId(1)

LABSTUDY>WRITE ##class(LabStudy.Sample).%ExistsId(1)
0
```

Linha por linha, com o **porquê**:

- `SET sample = ##class(LabStudy.Sample).%New()` — cria o objeto na memória. A variável `sample` recebe a **OREF**.
- `WRITE sample` → `1@LabStudy.Sample`. Isso confirma visualmente que `sample` guarda um bilhete, não um dado. O `1` é o número do bilhete nesta sessão.
- `WRITE sample.%Id()` — **não sai nada**. O objeto ainda não foi gravado, então ainda não tem ID. Este é o momento de fixar: **ID vem do `%Save()`, não do `%New()`**.
- `SET sample.Code = "S-0001"` — o **ponto** entre a variável e o nome da propriedade significa *"acesse este membro do objeto apontado por esta OREF"*. Repare que aqui não usamos `..`: os dois pontos são para dentro da própria classe; aqui estamos fora dela, com uma OREF na mão.
- `SET status = sample.%Save()` — grava. Guardamos o retorno numa variável, sempre.
- `WRITE $$$ISOK(status)` → `1` — confirma sucesso. Se tivesse dado erro, aqui sairia `0` e você usaria `DO $SYSTEM.Status.DisplayError(status)` para ver o motivo.
- `WRITE sample.%Id()` → `1` — **agora** tem ID.
- `SET sample = ""` — descarta a OREF de propósito, para provar o próximo passo. Quando nenhuma variável aponta mais para um objeto, o IRIS libera a memória dele sozinho.
- `SET reopened = ##class(LabStudy.Sample).%OpenId(1)` — vai ao disco e traz o objeto de volta. Repare que voltamos ao `##class(...)`: estamos pedindo **à classe** que abra.
- `WRITE reopened.Code, ...` — os dados continuam lá. É a prova de que a persistência funcionou.
- `WRITE ##class(LabStudy.Sample).%ExistsId(99)` → `0` — não existe objeto 99.
- `SET status = ##class(LabStudy.Sample).%DeleteId(1)` — apaga.
- O último `%ExistsId(1)` devolve `0`, confirmando que sumiu.

### 4.3 A mesma coisa pelo SQL

Sem escrever uma linha a mais, os dados estão numa tabela. No Portal, em **System Explorer → SQL**, no namespace `LABSTUDY`, rode:

```sql
SELECT ID, Code, Material FROM LabStudy.Sample
```

Você verá os objetos gravados como linhas, com a coluna `ID` correspondendo ao `%Id()`. É o mesmo dado, olhado por outra porta.

---

## 5. Variações e detalhes

### 5.1 Herança simples

```objectscript
Class LabStudy.BloodSample Extends LabStudy.Sample
{
Property Volume As %Integer;
}
```

`LabStudy.BloodSample` recebe **tudo** que `LabStudy.Sample` tem (as propriedades `Code` e `Material`, e toda a capacidade de persistir que veio de `%Persistent`) e acrescenta `Volume`.

A cadeia é: `%Persistent` → `LabStudy.Sample` → `LabStudy.BloodSample`. Cada elo herda de tudo que veio antes.

### 5.2 Herança múltipla e superclasse primária

O IRIS permite herdar de **mais de uma** classe:

```objectscript
Class LabStudy.Report Extends (%Persistent, LabStudy.Printable)
{
}
```

Regras que a prova cobra:

- Os parênteses são **obrigatórios** quando há mais de uma superclasse.
- A **primeira** da lista é a **superclasse primária** (*primary superclass*). Ela tem um papel especial: é dela que a classe herda o **armazenamento** (a forma como os dados são gravados em disco).
- Por isso, numa classe persistente com herança múltipla, **`%Persistent` (ou a classe persistente da qual você deriva) deve vir primeiro**.

E se duas superclasses tiverem um método com o mesmo nome? Quem ganha?

Isso é controlado pela palavra-chave **`Inheritance`**:

```objectscript
Class LabStudy.Report Extends (%Persistent, LabStudy.Printable) [ Inheritance = left ]
```

- **`Inheritance = left`** — é o **padrão**. Em caso de conflito, vence a classe mais à **esquerda** da lista.
- **`Inheritance = right`** — inverte: vence a mais à **direita**.

Repare no detalhe: `Inheritance` muda quem vence o conflito de nomes, mas **não** muda quem é a superclasse primária. A superclasse primária continua sendo a primeira da lista. Essa distinção é exatamente o tipo de pergunta que aparece na prova.

### 5.3 `Abstract` — a classe que não pode virar objeto

```objectscript
Class LabStudy.BaseEntity Extends %RegisteredObject [ Abstract ]
{
}
```

Uma classe **abstrata** é um molde incompleto, feito para ser herdado, nunca usado direto.

- Você **não pode** chamar `%New()` nela. Se tentar, recebe uma **OREF nula**.
- Ela serve para reunir o que é comum a várias classes filhas.
- Uma classe **persistente abstrata não gera tabela SQL** própria.

### 5.4 `Final` — a classe que não pode ter filhos

```objectscript
Class LabStudy.Constants Extends %RegisteredObject [ Final ]
{
}
```

Uma classe **final** não pode ser herdada por ninguém. Se outra classe tentar `Extends LabStudy.Constants`, a compilação falha.

`Abstract` e `Final` são opostos em espírito: uma **só** serve para ser herdada, a outra **não** pode ser herdada. E, logicamente, uma classe não pode ser as duas ao mesmo tempo de forma útil.

### 5.5 `SqlTableName` — mudando o nome da tabela

Por padrão, a tabela SQL tem o mesmo nome da classe. Para mudar:

```objectscript
Class LabStudy.Patient Extends %Persistent [ SqlTableName = PATIENT ]
```

Agora, no SQL, a tabela é `LabStudy.PATIENT`, embora a classe continue sendo `LabStudy.Patient`. Isso é útil quando o nome da classe conflita com uma palavra reservada do SQL, ou quando você precisa casar com um padrão de nomenclatura de banco já existente.

Há também a palavra-chave `SqlRowIdName`, para renomear a coluna do ID. Ambas são cosméticas do lado SQL e não mudam nada do lado dos objetos.

### 5.6 Descobrindo a classe de um objeto em tempo de execução

Três ferramentas, três usos:

```
LABSTUDY>SET s = ##class(LabStudy.Sample).%New()

LABSTUDY>WRITE $CLASSNAME(s)
LabStudy.Sample

LABSTUDY>WRITE s.%ClassName(1)
LabStudy.Sample

LABSTUDY>WRITE s.%ClassName()
Sample

LABSTUDY>WRITE $ISOBJECT(s)
1

LABSTUDY>WRITE $ISOBJECT("texto qualquer")
0
```

- **`$CLASSNAME(oref)`** — função do sistema; devolve o nome completo da classe do objeto.
- **`oref.%ClassName(1)`** — método de instância. O argumento `1` significa *"quero o nome completo, com pacote"*. Sem argumento, devolve **só o nome curto**. Essa diferença cai na prova.
- **`$ISOBJECT(x)`** — devolve `1` se `x` for uma OREF válida, `0` se for qualquer outra coisa (texto, número, vazio). É a forma correta e segura de testar se um `%OpenId()` deu certo.

### 5.7 Compilando e administrando classes pelo Terminal

Nem sempre você vai ter o VS Code aberto. Estes comandos valem ouro:

```
LABSTUDY>DO $SYSTEM.OBJ.Compile("LabStudy.Sample","ck")

LABSTUDY>DO $SYSTEM.OBJ.CompilePackage("LabStudy","ck")

LABSTUDY>DO $SYSTEM.OBJ.Delete("LabStudy.Sample")
```

- **`$SYSTEM.OBJ.Compile(nome, flags)`** — compila uma classe.
- **`$SYSTEM.OBJ.CompilePackage(pacote, flags)`** — compila todas as classes de um pacote.
- **`$SYSTEM.OBJ.Delete(nome)`** — apaga a **definição** da classe.
- As **flags** são letras que ajustam o comportamento. As mais comuns: `c` (compilar), `k` (manter o código-fonte gerado), `d` (mostrar detalhes na tela), `b` (compilar também as subclasses). A lista completa é longa: **verificar na documentação oficial** quando precisar de uma flag específica.

Um cuidado: apagar a **definição** de uma classe persistente não apaga necessariamente os **dados** já gravados por ela. Para limpar os dados, use `%KillExtent()` **antes** de apagar a classe.

### 5.8 `%RegisteredObject` na prática

Uma classe não persistente é útil o tempo todo. Exemplo:

```objectscript
/// Holds a temporary calculation result. Never stored on disk.
Class LabStudy.ResultBuffer Extends %RegisteredObject
{

Property Value As %Numeric;

Property Unit As %String;

}
```

```
LABSTUDY>SET buf = ##class(LabStudy.ResultBuffer).%New()
LABSTUDY>SET buf.Value = 13.5
LABSTUDY>SET buf.Unit = "g/dL"
LABSTUDY>WRITE buf.Value, " ", buf.Unit, !
13.5 g/dL
LABSTUDY>WRITE buf.%Save()
```

A última linha **dá erro**: `<METHOD DOES NOT EXIST>`, porque `%RegisteredObject` **não tem** `%Save()`. Essa classe não sabe gravar. Se você fechar a sessão, `buf` desaparece para sempre.

### 5.9 E o `%Close()`?

Em código antigo você vai encontrar `DO obj.%Close()`. Isso vinha da época do Caché, quando era preciso fechar objetos explicitamente para liberar memória.

No IRIS, **isso não é mais necessário**: quando nenhuma variável aponta mais para o objeto, o IRIS libera sozinho. A forma normal de "soltar" um objeto é simplesmente atribuir outra coisa à variável, ou deixá-la sair de escopo:

```
SET sample = ""
```

Saber reconhecer `%Close()` em código legado é útil; escrevê-lo em código novo, não.

---

## 6. Pegadinhas e erros comuns

**1) Achar que `%New()` grava.**
`%New()` só cria na memória. Sem `%Save()`, nada vai para o disco e o objeto não tem ID. Se você fechar o Terminal, o objeto some.

**2) Confundir o número da OREF com o ID.**
`WRITE obj` mostrando `3@LabStudy.Sample` **não** quer dizer que o ID é 3. Aquele número é um contador interno da sessão. O ID se lê com `obj.%Id()`.

**3) Confundir o que cada método devolve.**
- `%New()` → **OREF**
- `%OpenId()` → **OREF**
- `%Save()` → **%Status**
- `%DeleteId()` → **%Status**
- `%ExistsId()` → **1 ou 0**
- `%Id()` → **texto com o ID**

Trocar OREF por `%Status` é a pegadinha mais frequente deste tópico.

**4) Não conferir o retorno de `%OpenId()`.**
Se o ID não existe, você recebe `""` sem erro nenhum. O erro `<INVALID OREF>` só aparece depois, quando você tenta usar o objeto. Sempre teste:

```
SET p = ##class(LabStudy.Patient).%OpenId(id)
IF '$ISOBJECT(p) { WRITE "not found", ! QUIT }
```

O apóstrofo `'` antes de `$ISOBJECT` significa **negação**: "se **não** for objeto". Vamos formalizar os operadores lógicos no Capítulo 4.4.

**5) Chamar método de instância pela classe, ou o contrário.**
- `##class(LabStudy.Sample).%Save()` está **errado**: não existe "salvar a classe".
- `sample.%OpenId(1)` está **errado**: abrir é responsabilidade da classe.

Regra prática: se a operação precisa de um objeto **já existente**, é pela OREF. Se ela **cria ou localiza** um objeto, é pela classe.

**6) Esquecer o ponto e vírgula na declaração de propriedade.**
`Property Code As %String` sem o `;` não compila. Métodos, ao contrário, terminam com `}` e não levam `;`.

**7) Colocar `%Persistent` fora da primeira posição na herança múltipla.**
`Extends (LabStudy.Printable, %Persistent)` faz da `Printable` a superclasse primária, e o armazenamento sai errado ou não é gerado como esperado. Persistente vem **primeiro**.

**8) Tentar `%New()` numa classe abstrata.**
Não dá erro barulhento: devolve OREF nula. Você fica com `""` na mão e só descobre o problema depois. Mais um motivo para testar com `$ISOBJECT`.

**9) Esquecer que `%ClassName()` sem argumento devolve o nome curto.**
`s.%ClassName()` → `Sample`. `s.%ClassName(1)` → `LabStudy.Sample`.

**10) Achar que apagar a classe apaga os dados.**
`$SYSTEM.OBJ.Delete()` remove a definição. Os dados podem continuar em disco. Limpe com `%KillExtent()` antes, se for essa a intenção.

**11) Ignorar o `%Status` do `%Save()`.**
Um `%Save()` pode falhar silenciosamente do seu ponto de vista se você não olhar o retorno. Você acha que gravou; não gravou. Sempre guarde e sempre confira.

**12) Criar a classe no namespace errado.**
Se compilou em `USER` e está executando em `LABSTUDY`, o erro será `<CLASS DOES NOT EXIST>`. Confira `$NAMESPACE` e o `"ns"` do `settings.json`.

---

## 7. MÃO NA MASSA

> Lembre da **receita de execução**: IRIS ligado → Terminal → `ZN "LABSTUDY"` → escrever a classe no VS Code em `src/` → `Ctrl+S` para compilar → executar no Terminal.

---

### Exercício 1.1 — Diferenciando os três tipos de classe

**a) Enunciado:** Crie três classes no pacote `LabStudy.Demo`:

1. `Temp`, herdando de `%RegisteredObject`, com uma propriedade `Note As %String`.
2. `Stored`, herdando de `%Persistent`, com uma propriedade `Note As %String`.
3. Prove, no Terminal, que a primeira **não** sabe gravar e a segunda sabe. Prove também que só a segunda tem ID.

**b) Dica:** Tente chamar `%Save()` na primeira e observe qual erro aparece. Anote a mensagem exata.

**c) Como testar:** Na `Temp`, o `%Save()` deve resultar em `<METHOD DOES NOT EXIST>`. Na `Stored`, deve devolver `1` para `$$$ISOK` e um ID numérico.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

`src/LabStudy/Demo/Temp.cls`:

```objectscript
/// In-memory only. Cannot be stored on disk.
Class LabStudy.Demo.Temp Extends %RegisteredObject
{

Property Note As %String;

}
```

`src/LabStudy/Demo/Stored.cls`:

```objectscript
/// Persistent class. Gets an ID and an SQL table.
Class LabStudy.Demo.Stored Extends %Persistent
{

Property Note As %String;

}
```

No Terminal:

```
LABSTUDY>SET t = ##class(LabStudy.Demo.Temp).%New()
LABSTUDY>SET t.Note = "temporary"
LABSTUDY>WRITE t.Note, !
temporary

LABSTUDY>WRITE t.%Save()

WRITE t.%Save()
       ^
<METHOD DOES NOT EXIST> *%Save,LabStudy.Demo.Temp

LABSTUDY>SET s = ##class(LabStudy.Demo.Stored).%New()
LABSTUDY>SET s.Note = "stored"
LABSTUDY>SET sc = s.%Save()
LABSTUDY>WRITE $$$ISOK(sc), " id=", s.%Id(), !
1 id=1
```

**Por que cada decisão:**

- As duas classes têm **exatamente** a mesma propriedade. A única diferença é de quem herdam. Isso isola a variável do experimento: tudo que muda vem da superclasse.
- O erro `<METHOD DOES NOT EXIST>` prova o ponto central do capítulo: **capacidade vem da herança**. `%RegisteredObject` nunca recebeu a habilidade de gravar.
- O pacote `LabStudy.Demo` separa o material de experimento do projeto de verdade. Depois é fácil apagar o pacote inteiro sem tocar no resto.

---

### Exercício 1.2 — O ciclo de vida completo, com verificação de erro

**a) Enunciado:** Usando a classe `LabStudy.Demo.Stored` do exercício anterior, escreva no Terminal uma sequência que: crie três objetos com notas diferentes, grave cada um conferindo o `%Status`, imprima os IDs, reabra o segundo pelo ID e imprima sua nota, apague o segundo e prove que ele sumiu com `%ExistsId`.

**b) Dica:** Você vai repetir o mesmo bloco três vezes. Use nomes de variável diferentes ou reutilize a mesma variável — os dois funcionam, porque o `%Save()` já gravou antes de você reutilizar.

**c) Como testar:** Você deve ver três IDs em sequência, a nota correta do segundo objeto, e `0` no `%ExistsId` final.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

```
LABSTUDY>DO ##class(LabStudy.Demo.Stored).%KillExtent()

LABSTUDY>FOR i=1:1:3 { SET o = ##class(LabStudy.Demo.Stored).%New() SET o.Note = "note "_i SET sc = o.%Save() WRITE "saved id=", o.%Id(), " ok=", $$$ISOK(sc), ! }
saved id=1 ok=1
saved id=2 ok=1
saved id=3 ok=1

LABSTUDY>SET second = ##class(LabStudy.Demo.Stored).%OpenId(2)

LABSTUDY>WRITE $ISOBJECT(second), " -> ", second.Note, !
1 -> note 2

LABSTUDY>SET sc = ##class(LabStudy.Demo.Stored).%DeleteId(2)

LABSTUDY>WRITE $$$ISOK(sc), !
1

LABSTUDY>WRITE ##class(LabStudy.Demo.Stored).%ExistsId(2), !
0

LABSTUDY>SET gone = ##class(LabStudy.Demo.Stored).%OpenId(2)

LABSTUDY>WRITE $ISOBJECT(gone), !
0
```

**Por que cada decisão:**

- `%KillExtent()` no início limpa o que ficou do exercício anterior, para os IDs saírem previsíveis. **Atenção:** mesmo assim, se você já tinha gravado objetos antes, os IDs novos podem continuar a sequência antiga em vez de recomeçar do 1 — o contador de IDs é independente dos dados. Se os seus números saírem diferentes, está certo do mesmo jeito.
- `FOR i=1:1:3 { ... }` — laço que repete três vezes, com `i` valendo 1, 2 e 3. A estrutura completa de laços é o Capítulo 4.5; aqui use como receita.
- `"note "_i` — o sublinhado cola o texto com o número, formando `note 1`, `note 2`, `note 3`.
- `WRITE $ISOBJECT(second)` antes de usar `second.Note` — este é o hábito que evita `<INVALID OREF>`. Primeiro confirme que veio objeto, depois use.
- A última dupla de comandos mostra o comportamento silencioso do `%OpenId()`: ID inexistente devolve `0` em `$ISOBJECT`, sem erro nenhum.

---

### Exercício 1.3 — Herança

**a) Enunciado:** Crie `LabStudy.Demo.Base` (persistente) com a propriedade `CreatedBy As %String`. Crie `LabStudy.Demo.Child`, que herda de `Base` e acrescenta `Detail As %String`. No Terminal, crie um objeto de `Child`, preencha **as duas** propriedades, grave e reabra, provando que a propriedade herdada também foi persistida.

**b) Dica:** Você não precisa redeclarar `CreatedBy` na classe filha. Herdar significa que ela já está lá.

**c) Como testar:** Depois de reabrir pelo ID, as duas propriedades devem vir preenchidas.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

`src/LabStudy/Demo/Base.cls`:

```objectscript
/// Base persistent class for the inheritance exercise.
Class LabStudy.Demo.Base Extends %Persistent
{

Property CreatedBy As %String;

}
```

`src/LabStudy/Demo/Child.cls`:

```objectscript
/// Inherits CreatedBy from Base and adds its own property.
Class LabStudy.Demo.Child Extends LabStudy.Demo.Base
{

Property Detail As %String;

}
```

```
LABSTUDY>SET c = ##class(LabStudy.Demo.Child).%New()
LABSTUDY>SET c.CreatedBy = "Aziz"
LABSTUDY>SET c.Detail = "inherited property test"
LABSTUDY>SET sc = c.%Save()
LABSTUDY>SET id = c.%Id()
LABSTUDY>SET c = ""

LABSTUDY>SET back = ##class(LabStudy.Demo.Child).%OpenId(id)
LABSTUDY>WRITE back.CreatedBy, " | ", back.Detail, !
Aziz | inherited property test

LABSTUDY>WRITE back.%ClassName(1), !
LabStudy.Demo.Child

LABSTUDY>WRITE back.%ClassName(), !
Child
```

**Por que cada decisão:**

- `Child` **não** declara `Extends %Persistent`. Não precisa: `Base` já é persistente, e a capacidade desce pela cadeia.
- Guardar o ID numa variável (`SET id = c.%Id()`) antes de descartar a OREF é o padrão correto. Depois de `SET c = ""`, o objeto não está mais na sua mão — mas o ID continua sendo o endereço dele no armário.
- As duas chamadas de `%ClassName` no final servem para você **ver com os próprios olhos** a diferença entre passar `1` e não passar nada. Guarde essa diferença.

---

### Exercício 1.4 — Abstract na prática

**a) Enunciado:** Crie `LabStudy.Demo.Shape` como classe **abstrata** herdando de `%RegisteredObject`. Tente criar um objeto dela com `%New()` e observe o que acontece. Depois crie `LabStudy.Demo.Circle`, que herda de `Shape`, e prove que **essa** pode ser instanciada.

**b) Dica:** `%New()` numa classe abstrata não explode; ele devolve algo. Use `$ISOBJECT` para descobrir o quê.

**c) Como testar:** `$ISOBJECT` deve devolver `0` para a abstrata e `1` para a filha concreta.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

`src/LabStudy/Demo/Shape.cls`:

```objectscript
/// Abstract base. Cannot be instantiated directly.
Class LabStudy.Demo.Shape Extends %RegisteredObject [ Abstract ]
{

Property Label As %String;

}
```

`src/LabStudy/Demo/Circle.cls`:

```objectscript
/// Concrete subclass of the abstract Shape.
Class LabStudy.Demo.Circle Extends LabStudy.Demo.Shape
{

Property Radius As %Numeric;

}
```

```
LABSTUDY>SET s = ##class(LabStudy.Demo.Shape).%New()

LABSTUDY>WRITE $ISOBJECT(s), !
0

LABSTUDY>WRITE s = "", !
1

LABSTUDY>SET c = ##class(LabStudy.Demo.Circle).%New()

LABSTUDY>WRITE $ISOBJECT(c), !
1

LABSTUDY>SET c.Label = "small circle"
LABSTUDY>SET c.Radius = 2.5
LABSTUDY>WRITE c.Label, " r=", c.Radius, !
small circle r=2.5
```

**Por que cada decisão:**

- `[ Abstract ]` entre colchetes é a palavra-chave de classe. Colchetes vêm **depois** do `Extends` e **antes** da chave de abertura.
- O `%New()` da abstrata devolve OREF nula. `WRITE s = ""` devolvendo `1` prova que o retorno foi mesmo a string vazia. (No ObjectScript, o sinal `=` dentro de uma expressão é **comparação**, não atribuição — quem atribui é o comando `SET`. Isso é assunto do Capítulo 4.4, mas fica registrado desde já.)
- `Circle` herda `Label` de `Shape` mesmo `Shape` sendo abstrata. Abstrata não significa vazia: significa "não instanciável diretamente".

---

### Exercício 1.5 — PROJETO CONTÍNUO: nasce o paciente

Chegou a hora de dar o primeiro passo real no sistema de laboratório.

**a) Enunciado:** Crie a classe persistente `LabStudy.Patient`, com:

1. As propriedades `Name As %String`, `BirthDate As %Date` e `RecordNumber As %String`.
2. A palavra-chave de classe `SqlTableName = PATIENT`.
3. Um `ClassMethod` chamado `Create` que recebe nome, data de nascimento e número de registro, cria o objeto, grava e **devolve o ID** se deu certo, ou vazio se falhou (imprimindo o erro na tela).
4. Um `ClassMethod` chamado `Show` que recebe um ID, abre o paciente e imprime seus dados de forma legível — ou avisa que não encontrou.

Depois, atualize `LabStudy.App`: mude o parâmetro `VERSION` para `"0.2"` e acrescente um `ClassMethod` chamado `Status` que imprime quantos pacientes existem, usando `%ExistsId` num laço de 1 até 20 (uma forma provisória e ingênua — vamos fazer isso direito com SQL no Capítulo 4.6).

**b) Dica:**
- `%Date` é o tipo de data do IRIS. Internamente ele guarda um número, não um texto. Para converter de `"1990-05-17"` para o formato interno, use `$ZDATEH("1990-05-17", 3)`. Para converter de volta e mostrar na tela, `$ZDATE(valor, 3)`. O `3` é o código do formato ano-mês-dia. Datas são o Capítulo 4.4; aqui use como receita.
- Para devolver vazio em caso de falha, `QUIT ""`.
- Para imprimir o erro, `DO $SYSTEM.Status.DisplayError(sc)`.

**c) Como testar:**

```
LABSTUDY>SET id = ##class(LabStudy.Patient).Create("Maria Silva","1990-05-17","REG-001")
LABSTUDY>WRITE id, !
LABSTUDY>DO ##class(LabStudy.Patient).Show(id)
LABSTUDY>DO ##class(LabStudy.Patient).Show(999)
LABSTUDY>DO ##class(LabStudy.App).Status()
```

O `Show` com um ID válido deve imprimir os dados com a data legível. Com 999, deve avisar que não encontrou. O `Status` deve contar os pacientes.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

`src/LabStudy/Patient.cls`:

```objectscript
/// A patient of the LabStudy laboratory system.
/// This class grows along the study material.
Class LabStudy.Patient Extends %Persistent [ SqlTableName = PATIENT ]
{

/// Full name of the patient.
Property Name As %String;

/// Date of birth, stored in the internal date format.
Property BirthDate As %Date;

/// External record number used by the laboratory.
Property RecordNumber As %String;

/// Creates and saves a patient.
/// Returns the new ID on success, or an empty string on failure.
ClassMethod Create(name As %String, birthDate As %String, recordNumber As %String) As %String
{
    set patient = ..%New()
    set patient.Name = name
    set patient.BirthDate = $ZDATEH(birthDate, 3)
    set patient.RecordNumber = recordNumber

    set sc = patient.%Save()
    if $$$ISERR(sc) {
        write "Could not save patient:", !
        do $SYSTEM.Status.DisplayError(sc)
        quit ""
    }

    quit patient.%Id()
}

/// Prints the data of one patient identified by id.
ClassMethod Show(id As %String) As %Status
{
    set patient = ..%OpenId(id)
    if '$ISOBJECT(patient) {
        write "Patient not found: ", id, !
        quit $$$OK
    }

    write "------------------------------", !
    write "Id:     ", patient.%Id(), !
    write "Name:   ", patient.Name, !
    write "Birth:  ", $ZDATE(patient.BirthDate, 3), !
    write "Record: ", patient.RecordNumber, !
    write "------------------------------", !
    quit $$$OK
}

}
```

`src/LabStudy/App.cls` (versão atualizada):

```objectscript
/// Entry point and identity of the LabStudy laboratory system.
Class LabStudy.App Extends %RegisteredObject
{

/// Current version of the system.
Parameter VERSION = "0.2";

/// Human readable name of the system.
Parameter APPNAME = "LabStudy Laboratory System";

/// Prints identification data about the system.
ClassMethod About() As %Status
{
    write "==============================", !
    write ..#APPNAME, !
    write "Version: ", ..#VERSION, !
    write "Namespace: ", $NAMESPACE, !
    write "==============================", !
    quit $$$OK
}

/// Records that the environment was checked and shows the log.
ClassMethod CheckEnvironment() As %Status
{
    set ^LabStudyLog("environment", "checked") = 1
    set ^LabStudyLog("environment", "version") = ..#VERSION
    write "Environment log:", !
    zwrite ^LabStudyLog
    quit $$$OK
}

/// Naive patient count. Will be replaced by an SQL query later.
ClassMethod Status() As %Status
{
    set total = 0
    for id = 1:1:20 {
        if ##class(LabStudy.Patient).%ExistsId(id) {
            set total = total + 1
        }
    }
    write "Patients found (ids 1..20): ", total, !
    quit $$$OK
}

}
```

Execução esperada:

```
LABSTUDY>SET id = ##class(LabStudy.Patient).Create("Maria Silva","1990-05-17","REG-001")

LABSTUDY>WRITE id, !
1

LABSTUDY>DO ##class(LabStudy.Patient).Show(id)
------------------------------
Id:     1
Name:   Maria Silva
Birth:  1990-05-17
Record: REG-001
------------------------------

LABSTUDY>DO ##class(LabStudy.Patient).Show(999)
Patient not found: 999

LABSTUDY>DO ##class(LabStudy.App).Status()
Patients found (ids 1..20): 1
```

**Por que cada decisão:**

- **`..%New()` em vez de `##class(LabStudy.Patient).%New()`.** Estamos **dentro** da própria classe `LabStudy.Patient`, e os dois pontos significam "da minha própria classe". Além de mais curto, isso é mais robusto: se um dia essa classe for herdada, `..%New()` cria um objeto da **classe correta**, e não sempre um `Patient`.
- **`Create` devolve `%String` e não `%Status`.** Aqui a informação útil para quem chama é o **ID**. Devolver vazio como sinal de falha é um padrão simples e legítimo. Note que essa é uma escolha de projeto: em código mais rigoroso você devolveria o `%Status` e receberia o ID por referência. Vamos ver essa técnica no Capítulo 1.3.
- **`if $$$ISERR(sc) { ... quit "" }` — saída antecipada.** Tratar o erro e sair logo deixa o caminho feliz do método sem aninhamento. É um hábito que paga dividendos quando os métodos crescem.
- **`$ZDATEH` na entrada, `$ZDATE` na saída.** O `%Date` guarda um número interno. Se você fizesse `SET patient.BirthDate = "1990-05-17"`, o `%Save()` recusaria na validação, porque o texto não é uma data no formato interno. Converter na entrada e na saída é a forma correta.
- **`'$ISOBJECT(patient)` antes de usar.** Sem essa linha, `Show(999)` daria `<INVALID OREF>` em vez de uma mensagem clara. Esta é a diferença entre um sistema que quebra e um sistema que explica.
- **`Show` devolve `$$$OK` mesmo quando não encontra.** "Não encontrado" não é um erro do sistema; é um resultado normal de uma consulta. Erro seria o banco estar fora do ar. Essa distinção importa quando você começa a encadear chamadas.
- **`Status` com laço até 20 é propositalmente ingênuo.** Ele funciona, ensina `%ExistsId` e é honesto sobre sua limitação no comentário. No Capítulo 4.6 vamos substituí-lo por uma consulta SQL de verdade — e você vai sentir na pele a diferença.
- **`SqlTableName = PATIENT`.** Além de exercitar a palavra-chave, deixa a tabela com um nome em maiúsculas, no estilo que muitos bancos usam. Confira no Portal: **System Explorer → SQL** e rode `SELECT * FROM LabStudy.PATIENT`.

---

## 8. Quiz do capítulo

**Q1.** O que o método `%New()` devolve?

- A) Um `%Status`.
- B) Uma OREF, ou OREF nula em caso de falha.
- C) O ID do novo objeto.
- D) O nome da classe.

---

**Q2.** Considere:

```
SET p = ##class(LabStudy.Patient).%New()
SET p.Name = "Ana"
WRITE p.%Id()
```

O que aparece na tela?

- A) `1`
- B) `Ana`
- C) Nada, porque o objeto ainda não foi gravado.
- D) Erro `<METHOD DOES NOT EXIST>`.

---

**Q3.** Uma classe herda de `%RegisteredObject`. Quais operações estão disponíveis?

- A) `%New()` apenas; não há `%Save()` nem `%OpenId()`.
- B) `%New()`, `%Save()` e `%OpenId()`.
- C) Apenas `%Save()`.
- D) Nenhuma; `%RegisteredObject` é abstrata.

---

**Q4.** Você quer representar um endereço que só existe dentro da ficha de um paciente, sem ID próprio e sem tabela própria. De qual classe ele deve herdar?

- A) `%Persistent`
- B) `%RegisteredObject`
- C) `%SerialObject`
- D) `%Library.DynamicObject`

---

**Q5.** Qual afirmação sobre `%OpenId()` com um ID inexistente está correta?

- A) Lança o erro `<INVALID OREF>` imediatamente.
- B) Devolve uma OREF nula, sem erro.
- C) Devolve `0`.
- D) Cria um objeto novo automaticamente.

---

**Q6.** Considere a declaração:

```
Class LabStudy.Report Extends (LabStudy.Printable, %Persistent)
```

Qual é o problema?

- A) Herança múltipla não é permitida no IRIS.
- B) Faltam colchetes em vez de parênteses.
- C) A superclasse primária passou a ser `LabStudy.Printable`, e o armazenamento persistente não é herdado como esperado.
- D) Não há problema algum.

---

**Q7.** O que a palavra-chave `Inheritance = right` faz?

- A) Torna a última classe da lista a superclasse primária.
- B) Inverte a ordem de resolução em caso de membros com o mesmo nome, fazendo vencer a classe mais à direita.
- C) Impede que a classe seja herdada.
- D) Faz a classe herdar apenas de uma superclasse.

---

**Q8.** O que acontece ao chamar `%New()` numa classe marcada como `[ Abstract ]`?

- A) Erro de compilação.
- B) Erro `<CLASS DOES NOT EXIST>` em tempo de execução.
- C) Devolve uma OREF nula.
- D) Funciona normalmente.

---

**Q9.** Dado `SET s = ##class(LabStudy.Sample).%New()`, o que `WRITE s.%ClassName()` imprime?

- A) `LabStudy.Sample`
- B) `Sample`
- C) `%Persistent`
- D) `1@LabStudy.Sample`

---

**Q10.** Você vê no Terminal:

```
LABSTUDY>WRITE obj
5@LabStudy.Patient
```

O que o número `5` significa?

- A) O ID do objeto no banco de dados.
- B) O número de propriedades da classe.
- C) Um contador interno de referências da sessão; não é o ID.
- D) A versão da classe compilada.

---

**Q11.** Qual sequência apaga corretamente do disco o objeto persistente de ID 7 da classe `LabStudy.Sample`?

- A) `SET o = ##class(LabStudy.Sample).%OpenId(7) DO o.%DeleteId()`
- B) `SET sc = ##class(LabStudy.Sample).%DeleteId(7)`
- C) `SET sc = ##class(LabStudy.Sample).%Delete(7)`
- D) `DO ##class(LabStudy.Sample).%KillExtent(7)`

---

**Q12.** Uma classe persistente `LabStudy.Patient` foi compilada com sucesso. Qual afirmação sobre SQL está correta?

- A) É preciso executar `CREATE TABLE` manualmente para acessar por SQL.
- B) A tabela SQL é criada automaticamente, com o pacote como schema e a classe como nome da tabela.
- C) Classes persistentes não são acessíveis por SQL.
- D) A tabela só é criada depois do primeiro `%Save()`.

---

### Gabarito comentado

**Q1 — Resposta: B.**
- **B está certa:** `%New()` constrói o objeto na memória e devolve a referência a ele; se não puder construir, devolve OREF nula (`""`).
- **A está errada:** quem devolve `%Status` é `%Save()` e `%DeleteId()`.
- **C está errada:** o ID só existe após um `%Save()` bem-sucedido, e se obtém com `%Id()`.
- **D está errada:** nome de classe se obtém com `$CLASSNAME()` ou `%ClassName()`.

**Q2 — Resposta: C.**
- **C está certa:** o objeto foi criado e preenchido, mas não gravado. Sem `%Save()`, não há ID, e `%Id()` devolve vazio.
- **A está errada:** o `1` só apareceria depois de gravar.
- **B está errada:** `%Id()` não devolve propriedades.
- **D está errada:** `%Id()` existe em classes persistentes; a chamada é válida.

**Q3 — Resposta: A.**
- **A está certa:** `%RegisteredObject` dá existência em memória. Persistência (`%Save`, `%OpenId`, `%DeleteId`, ID, tabela SQL) vem de `%Persistent`.
- **B está errada:** esses métodos não existem em `%RegisteredObject` — a chamada resulta em `<METHOD DOES NOT EXIST>`.
- **C está errada:** `%Save()` é justamente o que falta.
- **D está errada:** `%RegisteredObject` pode ser instanciada.

**Q4 — Resposta: C.**
- **C está certa:** `%SerialObject` existe para agrupar campos que são gravados **dentro** de um objeto persistente que os contém, sem ID nem tabela próprios.
- **A está errada:** `%Persistent` daria ao endereço ID e tabela próprios, que é exatamente o que não se quer.
- **B está errada:** `%RegisteredObject` não é gravado em disco de forma alguma.
- **D está errada:** `%Library.DynamicObject` serve para estruturas JSON dinâmicas, assunto do Capítulo 1.4.

**Q5 — Resposta: B.**
- **B está certa:** o método é silencioso. Devolve `""` e cabe a você testar com `$ISOBJECT`.
- **A está errada:** `<INVALID OREF>` só aparece mais tarde, quando você tenta usar a referência vazia.
- **C está errada:** quem devolve `0` ou `1` é `%ExistsId()`.
- **D está errada:** `%OpenId()` nunca cria objetos.

**Q6 — Resposta: C.**
- **C está certa:** a primeira classe da lista é a superclasse primária, e é dela que vem o armazenamento. Numa classe persistente, a classe persistente deve vir primeiro.
- **A está errada:** o IRIS permite herança múltipla.
- **B está errada:** parênteses são a sintaxe correta para múltiplas superclasses; colchetes são para palavras-chave.
- **D está errada:** há problema, e ele é de ordem.

**Q7 — Resposta: B.**
- **B está certa:** `Inheritance` controla a ordem de resolução de membros com o mesmo nome. `right` faz vencer a classe mais à direita.
- **A está errada:** a superclasse primária continua sendo a primeira da lista, independentemente de `Inheritance`.
- **C está errada:** quem impede herança é `Final`.
- **D está errada:** `Inheritance` não limita o número de superclasses.

**Q8 — Resposta: C.**
- **C está certa:** a chamada não explode; devolve OREF nula, e o problema só aparece quando você tenta usar o resultado.
- **A está errada:** a classe abstrata compila normalmente; o problema é de instanciação, em tempo de execução.
- **B está errada:** a classe existe; o que não se pode é instanciá-la.
- **D está errada:** é justamente o que `Abstract` proíbe.

**Q9 — Resposta: B.**
- **B está certa:** sem argumento, `%ClassName()` devolve o nome **curto**, sem o pacote.
- **A está errada:** o nome completo sai com `%ClassName(1)` ou `$CLASSNAME(s)`.
- **C está errada:** ele devolve a classe do objeto, não a superclasse.
- **D está errada:** a forma `n@Classe` é o que aparece ao escrever a OREF inteira, não ao chamar `%ClassName()`.

**Q10 — Resposta: C.**
- **C está certa:** o número antes do arroba é um contador interno de referências da sessão. O ID do banco se obtém com `%Id()`.
- **A está errada:** essa é a confusão clássica que a questão testa.
- **B está errada:** não tem relação com propriedades.
- **D está errada:** não tem relação com versão de compilação.

**Q11 — Resposta: B.**
- **B está certa:** `%DeleteId()` é chamado pela classe, recebe o ID e devolve `%Status`.
- **A está errada:** `%DeleteId()` não é método de instância; não se chama a partir da OREF dessa forma.
- **C está errada:** `%Delete()` espera um **OID**, não um ID simples.
- **D está errada:** `%KillExtent()` apaga **todos** os objetos da classe e não recebe um ID.

**Q12 — Resposta: B.**
- **B está certa:** a projeção SQL é automática na compilação de uma classe persistente: pacote vira schema, classe vira tabela, propriedades viram colunas.
- **A está errada:** não é preciso `CREATE TABLE`.
- **C está errada:** o acesso multimodelo (objeto e SQL sobre o mesmo dado) é característica central do IRIS.
- **D está errada:** a tabela existe a partir da compilação, mesmo vazia.

---

## 9. Resumo relâmpago

1. **Classe** é o molde; **objeto** é a peça; **OREF** é o bilhete que aponta para a peça na memória.
2. `%RegisteredObject` = só memória. `%Persistent` = disco, ID e tabela SQL. `%SerialObject` = pedaço embutido dentro de outro objeto.
3. `%RegisteredObject` é a base de todas; capacidade vem da herança.
4. Ciclo de vida: `%New()` → preencher → `%Save()` → `%OpenId()` → `%DeleteId()`.
5. `%New()` e `%OpenId()` devolvem **OREF**. `%Save()` e `%DeleteId()` devolvem **%Status**. `%ExistsId()` devolve **1/0**. `%Id()` devolve o **ID**.
6. **O ID só existe depois de um `%Save()` bem-sucedido.** IDs não são reaproveitados.
7. `n@Package.Class` é a exibição de uma OREF; aquele `n` **não** é o ID.
8. `%OpenId()` com ID inexistente devolve `""` **em silêncio**. Sempre teste com `$ISOBJECT()`.
9. Métodos que **criam ou localizam** são chamados pela classe (`##class(...)`); métodos que **agem sobre um objeto existente** são chamados pela OREF.
10. Herança múltipla usa parênteses; a **primeira** classe da lista é a **superclasse primária** e define o armazenamento.
11. `Inheritance = left` (padrão) ou `right` decide quem vence em conflito de nomes — e **não** muda a superclasse primária.
12. `[ Abstract ]` = não pode ser instanciada (`%New()` devolve OREF nula). `[ Final ]` = não pode ser herdada.
13. `[ SqlTableName = X ]` renomeia a tabela SQL sem mudar o nome da classe.
14. `$CLASSNAME(oref)` e `oref.%ClassName(1)` dão o nome completo; `oref.%ClassName()` dá o nome curto.
15. Toda classe persistente compilada ganha **automaticamente** uma tabela SQL: pacote = schema, classe = tabela, propriedades = colunas.
16. Sempre guarde e confira o `%Status`: `$$$ISOK`, `$$$ISERR`, `$SYSTEM.Status.DisplayError()`.
17. `%KillExtent()` apaga todos os dados da classe; use com muito cuidado.

---

## 10. Cartões de memorização

**Frente:** O que `%New()` devolve?
**Verso:** Uma OREF (referência ao objeto na memória), ou OREF nula `""` se não puder criar.

**Frente:** O que `%Save()` devolve?
**Verso:** Um `%Status`. Nunca uma OREF.

**Frente:** Quando um objeto persistente ganha ID?
**Verso:** Só depois de um `%Save()` bem-sucedido.

**Frente:** O que `%OpenId()` faz quando o ID não existe?
**Verso:** Devolve OREF nula `""`, em silêncio, sem erro.

**Frente:** Como testar se uma OREF é válida?
**Verso:** `$ISOBJECT(oref)` — devolve 1 se for objeto, 0 caso contrário.

**Frente:** Em `3@LabStudy.Patient`, o que é o `3`?
**Verso:** Um contador interno de referências da sessão. **Não** é o ID do banco.

**Frente:** Diferença entre `%RegisteredObject` e `%Persistent`.
**Verso:** `%RegisteredObject` vive só na memória. `%Persistent` grava em disco, tem ID e ganha tabela SQL.

**Frente:** Para que serve `%SerialObject`?
**Verso:** Para objetos embutidos: agrupam campos gravados **dentro** de um objeto persistente que os contém, sem ID nem tabela próprios.

**Frente:** Como se declara herança múltipla?
**Verso:** `Extends (ClasseA, ClasseB)` — com parênteses. A primeira é a superclasse primária.

**Frente:** O que define a superclasse primária e por que ela importa?
**Verso:** É a **primeira** da lista de `Extends`. É dela que a classe herda o armazenamento.

**Frente:** O que faz `Inheritance = right`?
**Verso:** Em conflito de membros com o mesmo nome, faz vencer a classe mais à direita. Não altera a superclasse primária.

**Frente:** O que acontece ao chamar `%New()` numa classe `Abstract`?
**Verso:** Devolve OREF nula. Não gera erro imediato.

**Frente:** O que faz `[ Final ]`?
**Verso:** Impede que a classe seja herdada.

**Frente:** `%ClassName()` com e sem argumento.
**Verso:** Com `1`, devolve o nome completo com pacote. Sem argumento, devolve só o nome curto.

**Frente:** Qual a diferença entre `%DeleteId()` e `%Delete()`?
**Verso:** `%DeleteId()` recebe um **ID**; `%Delete()` recebe um **OID**.

**Frente:** O que faz `%KillExtent()`?
**Verso:** Apaga **todos** os objetos da classe, sem as verificações normais. Perigoso.

**Frente:** Qual comando compila uma classe pelo Terminal?
**Verso:** `DO $SYSTEM.OBJ.Compile("Pacote.Classe","ck")`.

**Frente:** Como se descobre o motivo de um `%Status` de erro?
**Verso:** `DO $SYSTEM.Status.DisplayError(sc)` para mostrar na tela, ou `$SYSTEM.Status.GetErrorText(sc)` para obter o texto.

**Frente:** Como uma classe persistente vira tabela SQL?
**Verso:** Automaticamente na compilação: pacote = schema, classe = tabela, propriedades = colunas, objetos = linhas.

**Frente:** Dentro de um método da própria classe, como criar um objeto dela?
**Verso:** `set obj = ..%New()` — os dois pontos significam "da minha própria classe".

---

Digite CONTINUAR para o próximo capítulo.
