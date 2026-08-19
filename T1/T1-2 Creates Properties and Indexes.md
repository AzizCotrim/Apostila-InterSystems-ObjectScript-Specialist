# Apostila InterSystems ObjectScript Specialist
## Capítulo 2 — T1.2 Creates Properties and Indexes (Propriedades e índices)

> Continuamos no domínio **T1 — Manages Data Model** (23 questões de 76). Este é o capítulo mais longo até aqui, porque propriedades e índices são a espinha dorsal de qualquer modelo de dados no IRIS.

---

## 1. Objetivo do capítulo

Ao terminar este capítulo, você será capaz de:

1. Declarar **propriedades** de qualquer tipo, entendendo cada parte da sintaxe.
2. Usar os **tipos de dados** do IRIS: `%String`, `%Integer`, `%Numeric`, `%Boolean`, `%Date`, `%Time`, `%TimeStamp` e outros.
3. Configurar **parâmetros de tipo**: `MAXLEN`, `MINVAL`, `MAXVAL`, `VALUELIST`, `DISPLAYLIST`, `TRUNCATE`, `PATTERN`, `SCALE`.
4. Usar as **palavras-chave de propriedade**: `Required`, `InitialExpression`, `Calculated`, `Transient`, `ReadOnly`, `Private`, `SqlFieldName`, `MultiDimensional`.
5. Entender os **três formatos de um valor** — lógico, de exibição e ODBC — e converter entre eles.
6. Escrever **propriedades calculadas** com métodos `Get` e `Set` próprios.
7. Declarar **coleções**: `list Of` e `array Of`, e manipulá-las com `Insert`, `SetAt`, `GetAt`, `Count`, `RemoveAt`.
8. Embutir um **`%SerialObject`** dentro de uma classe persistente.
9. Criar **relacionamentos** um-para-muitos com `Relationship`, `Cardinality` e `Inverse`.
10. Criar **índices**: comuns, `Unique`, `IdKey`, `PrimaryKey` e `bitmap` — e saber quando cada um serve.
11. Reconstruir índices com **`%BuildIndices()`** e usar os **métodos gerados** por um índice único.
12. Validar objetos com **`%ValidateObject()`** e ler as mensagens de erro de validação.
13. Evoluir o projeto: `LabStudy.Patient` completo, com endereço embutido, alergias em lista, índices e relacionamento com `LabStudy.Exam`.

---

## 2. O conceito em linguagem de gente

### 2.1 Propriedade é campo do formulário

No Capítulo 1, a classe era o formulário em branco. As **propriedades** são os **campos impressos** nesse formulário: a linha do nome, o quadradinho da data de nascimento, o retângulo do número de registro.

Um formulário bem feito não tem só linhas em branco. Ele tem regras impressas ao lado de cada campo:

- *"Nome: (obrigatório)"*
- *"Data de nascimento: __/__/____"* — só cabe data, não cabe um bilhete de amor
- *"Sexo: ( ) M ( ) F"* — só duas opções, marque uma
- *"Observações: até 500 caracteres"*

Essas regras impressas são exatamente o que os **parâmetros de tipo** e as **palavras-chave de propriedade** fazem no IRIS. Você não escreve código para validar cada campo: você **declara a regra junto do campo**, e o IRIS obriga.

Esta é a ideia central do capítulo, e vale a pena parar nela: **no IRIS, a validação mora na definição do modelo, não espalhada no meio do código**. Quando você declara `Property Name As %String [ Required ]`, todo `%Save()` do sistema inteiro passa a exigir o nome. Você não precisa lembrar de checar em cada lugar.

### 2.2 Tipo é o formato do campo

Um campo de data e um campo de texto não são a mesma coisa. O tipo diz três coisas:

1. **O que cabe ali** — se você tentar colocar `"azul"` num campo `%Integer`, o IRIS recusa.
2. **Como o valor é guardado por dentro** — e aqui vem uma surpresa que explicaremos já já.
3. **Como o valor é mostrado para uma pessoa** — que pode ser bem diferente de como ele é guardado.

### 2.3 Os três rostos do mesmo valor

Esta é a ideia mais escorregadia do capítulo, e cai bastante na prova. Vamos com uma analogia.

Pense num arquivista antigo, meticuloso. Para economizar espaço no armário, ele **não** anota datas como "17 de maio de 1990". Ele criou um sistema: escolheu um dia zero lá no passado e passou a anotar só **quantos dias se passaram desde esse dia zero**. Assim, "17 de maio de 1990" vira, na ficha, um número como `54683`.

Quando alguém pede a ficha, ele consulta a tabela de conversão e diz em voz alta: "17 de maio de 1990". Quando outro sistema pede os dados por um cabo, ele manda no formato padronizado "1990-05-17".

O mesmo valor, três rostos:

| Rosto | Nome oficial | Como é o `%Date` do exemplo |
|---|---|---|
| Como o IRIS guarda | valor **lógico** (*logical*) | `54683` |
| Como uma pessoa lê na tela | valor de **exibição** (*display*) | `05/17/1990` (varia com o idioma configurado) |
| Como outro sistema recebe | valor **ODBC** | `1990-05-17` |

**O valor lógico é o que fica na propriedade.** Quando você faz `WRITE patient.BirthDate`, sai o número. Isso assusta na primeira vez, mas é proposital: comparar e ordenar números é rápido e sem ambiguidade.

E funciona assim para vários tipos:

- **`%Date`** — lógico é um **número inteiro** de dias contados a partir de uma data-base interna do IRIS.
- **`%Time`** — lógico é o **número de segundos** desde a meia-noite. `3600` é uma hora da manhã.
- **`%TimeStamp`** — aqui o lógico já é legível: o texto no formato `AAAA-MM-DD HH:MM:SS`.
- **`%Boolean`** — lógico é `1` (verdadeiro) ou `0` (falso).
- **`%String`** — lógico é o próprio texto. Sem mistério.

Cada tipo do IRIS sabe converter entre os três rostos. Voltaremos a isso com a sintaxe exata na seção 3.

### 2.4 Índice é o dedo indicador do arquivista

Imagine 200 mil fichas de pacientes no armário, guardadas por número de registro. Alguém pergunta: *"quem é a paciente chamada Maria Silva?"*

Sem ajuda nenhuma, o arquivista precisa abrir **todas** as fichas, uma por uma, até achar. Isso é o que se chama de varredura completa. Funciona, mas demora horrivelmente.

Agora imagine que, ao lado do armário, existe um **fichário alfabético**: uma caixinha com cartões pequenos, ordenados por nome, e cada cartão diz apenas *"Maria Silva → ficha 4711"*. O arquivista vai direto ao cartão, lê o número e pega a ficha certa.

Esse fichário é um **índice**.

Fatos que decorrem naturalmente da analogia — e que são exatamente o que a prova cobra:

- O índice **não guarda a ficha**, guarda um atalho para ela.
- Ele ocupa **espaço extra** no armário.
- Toda vez que uma ficha é criada, alterada ou apagada, **o fichário precisa ser atualizado também**. Ou seja: índice acelera a leitura e **desacelera um pouco a escrita**.
- Se você montar o fichário **depois** que o armário já está cheio, precisa passar por todas as fichas uma vez para criar os cartões. No IRIS isso se chama **reconstruir os índices** (`%BuildIndices()`).
- Um fichário pode ter a regra *"não pode haver dois cartões com o mesmo nome"*. Isso é um índice **único** (`Unique`).

### 2.5 Coleção é um campo que aceita vários valores

Alguns campos do formulário precisam aceitar mais de uma resposta. "Alergias conhecidas: ______" pode ter zero, uma ou sete respostas.

O IRIS oferece duas formas:

- **Lista** (`list Of`) — os valores ficam **em ordem**, numerados de 1 em diante. Como uma lista de compras: item 1, item 2, item 3.
- **Array** (`array Of`) — cada valor tem uma **etiqueta** (chave) que você escolhe. Como um chaveiro com etiquetas: a chave "casa", a chave "carro". A ordem é a ordem alfabética das etiquetas, não a ordem em que você inseriu.

Escolha simples: **se o que importa é a ordem, use lista. Se o que importa é achar pelo nome, use array.**

### 2.6 Objeto embutido versus relacionamento

Duas formas de uma classe conter outra, e a diferença importa muito.

**Embutido (`%SerialObject`):** o endereço do paciente. Ele não tem vida própria, não tem ficha separada, ninguém procura "o endereço 17". Ele é gravado **dentro** da ficha do paciente, como um carimbo. Se a ficha do paciente sumir, o endereço vai junto — e isso está certo.

**Relacionamento:** os exames do paciente. Cada exame **tem** vida própria: tem ficha, tem número, você pode listar todos os exames de hoje sem saber de quem são. Mas cada exame **pertence** a um paciente, e o paciente **tem vários** exames.

Regra de bolso: se a coisa nunca é procurada sozinha, embuta. Se ela é uma entidade de verdade, relacione.

---

## 3. A sintaxe explicada

### 3.1 A forma geral de uma propriedade

```
Property NomeDaPropriedade As Tipo(ParametrosDeTipo) [ PalavrasChave ] ;
```

Pedaço por pedaço:

- **`Property`** — palavra fixa que anuncia o membro. **Obrigatória.**
- **`NomeDaPropriedade`** — o nome. **Obrigatório.** Diferencia maiúsculas de minúsculas. Comece com letra.
- **`As Tipo`** — o tipo de dado. **Opcional na escrita, mas sempre presente na prática.** Se você omitir, o IRIS assume `%String`.
- **`(ParametrosDeTipo)`** — ajustes do tipo, entre parênteses, separados por vírgula. **Opcional.** Exemplo: `%String(MAXLEN = 100)`.
- **`[ PalavrasChave ]`** — características da propriedade, entre colchetes, separadas por vírgula. **Opcional.** Exemplo: `[ Required, InitialExpression = 0 ]`.
- **`;`** — ponto e vírgula final. **Obrigatório.** Esquecer isso é erro de compilação.

Repare na divisão de trabalho, que confunde muita gente:

- **Parênteses** = parâmetros **do tipo**. Falam sobre o *conteúdo*: qual o tamanho máximo, quais valores são aceitos.
- **Colchetes** = palavras-chave **da propriedade**. Falam sobre o *comportamento*: é obrigatória, é calculada, tem valor inicial.

Um exemplo com os dois ao mesmo tempo:

```objectscript
Property Name As %String(MAXLEN = 100) [ Required ];
```

Leia: *"propriedade Name, do tipo texto com no máximo 100 caracteres, obrigatória."*

### 3.2 Os tipos de dados principais

| Tipo | Guarda | Observação importante |
|---|---|---|
| `%String` | texto | **Tamanho máximo padrão é 50 caracteres.** Ajuste com `MAXLEN`. |
| `%Integer` | número inteiro | Aceita negativo. Use `MINVAL`/`MAXVAL`. |
| `%Numeric` | número com casas decimais fixas | Use `SCALE` para o número de casas. |
| `%Float` | número de ponto flutuante | Para grandezas científicas; pode ter imprecisão de arredondamento. |
| `%Boolean` | verdadeiro/falso | Lógico é `1` ou `0`. |
| `%Date` | data (sem hora) | Lógico é um **número inteiro de dias**. |
| `%Time` | hora (sem data) | Lógico é o **número de segundos** desde a meia-noite. |
| `%TimeStamp` | data e hora juntas | Lógico é o texto `AAAA-MM-DD HH:MM:SS`. |
| `%Currency` | valor monetário | Para dinheiro, evita problemas de arredondamento. |
| `%Name` | nome de pessoa | Um `%String` com validação de formato "Sobrenome,Nome". |
| `%Text` | texto longo, indexável por palavras | Para busca textual. |
| `%Binary` | dados binários | |
| `%List` | lista serializada | Estrutura interna do IRIS, vista no Capítulo 4.2. |

**Fixe este ponto:** `%String` sem `MAXLEN` aceita **50 caracteres**. Se você guardar 80, o `%Save()` falha com erro de validação. Isso é campeão de pegadinha.

### 3.3 Parâmetros de tipo

```objectscript
Property Note As %String(MAXLEN = 500);
Property Age As %Integer(MINVAL = 0, MAXVAL = 130);
Property Weight As %Numeric(SCALE = 2);
Property Sex As %String(VALUELIST = ",M,F");
Property Code As %String(PATTERN = "3N1""-""4N");
Property Comment As %String(MAXLEN = 20, TRUNCATE = 1);
```

- **`MAXLEN`** — comprimento máximo do texto. Ultrapassar causa erro de validação.
- **`MINVAL` / `MAXVAL`** — valor mínimo e máximo permitidos para números.
- **`SCALE`** — quantas casas decimais são mantidas. `SCALE = 2` arredonda `3.14159` para `3.14`.
- **`TRUNCATE`** — se valer `1`, em vez de dar erro por texto grande demais, o IRIS **corta** no `MAXLEN`. Cuidado: isso perde dados silenciosamente.
- **`VALUELIST`** — a lista dos únicos valores permitidos. Sintaxe peculiar: **o primeiro caractere da string é o separador**. Em `",M,F"`, o separador é a vírgula, e os valores permitidos são `M` e `F`. Você poderia escrever `"|M|F"` e o separador seria a barra vertical.
- **`DISPLAYLIST`** — os rótulos legíveis, na **mesma ordem** do `VALUELIST`. Serve para o valor lógico ser `M` e a exibição ser `Masculino`.
- **`PATTERN`** — um padrão que o texto deve casar, usando a sintaxe de padrões do ObjectScript (vista no Capítulo 4.3).

Exemplo do par `VALUELIST` + `DISPLAYLIST`:

```objectscript
Property Status As %String(VALUELIST = ",P,D,C", DISPLAYLIST = ",Pending,Done,Cancelled");
```

Aqui, o que fica gravado é `P`, `D` ou `C` (curto, econômico), mas a exibição mostra a palavra inteira.

### 3.4 Palavras-chave de propriedade

```objectscript
Property Name As %String [ Required ];
Property Active As %Boolean [ InitialExpression = 1 ];
Property CreatedOn As %TimeStamp [ InitialExpression = {$ZDATETIME($HOROLOG, 3)} ];
Property Age As %Integer [ Calculated ];
Property TempScore As %Integer [ Transient ];
Property Secret As %String [ Private ];
Property Serial As %String [ ReadOnly ];
Property Notes As %String [ SqlFieldName = OBSERVATIONS ];
```

- **`Required`** — não pode ficar vazia. O `%Save()` recusa e devolve um `%Status` de erro dizendo qual campo faltou.
- **`InitialExpression`** — o valor que a propriedade recebe automaticamente no `%New()`. Pode ser um literal (`1`, `"pending"`) ou uma expressão ObjectScript **entre chaves** (`{$HOROLOG}`). As chaves são a forma de dizer ao compilador *"isto é código para calcular, não um texto literal"*.
- **`Calculated`** — a propriedade **não ocupa espaço nenhum**: nem em disco, nem na memória. O valor é produzido sob demanda por um método `Get` que você escreve. Toda leitura chama o método.
- **`Transient`** — a propriedade **existe na memória** mas **não é gravada** em disco. Útil para valores intermediários que acompanham o objeto durante a execução.
- **`Private`** — só pode ser acessada de dentro da própria classe. De fora, tentar `obj.Secret` dá erro.
- **`ReadOnly`** — só pode receber valor pelo `InitialExpression` ou de dentro da classe; não pode ser alterada de fora com `SET`.
- **`SqlFieldName`** — muda o nome da **coluna** SQL sem mudar o nome da propriedade.
- **`MultiDimensional`** — a propriedade se comporta como uma estrutura com subscritos (vista no Capítulo 4.1). Não é gravada em disco automaticamente e não aparece no SQL.

Diferença que cai na prova: **`Calculated` não tem espaço em memória; `Transient` tem espaço em memória mas não em disco.**

### 3.5 Propriedades calculadas: os métodos `Get` e `Set`

Para cada propriedade `X`, o IRIS reconhece dois métodos de nome fixo:

- **`XGet()`** — chamado toda vez que alguém **lê** `obj.X`.
- **`XSet(value)`** — chamado toda vez que alguém **escreve** `obj.X = valor`.

Se você não escrever nenhum dos dois, o IRIS usa o comportamento padrão (guardar e devolver o valor). Se você escrever, o **seu** método assume.

```objectscript
Property BirthDate As %Date;

/// Age in whole years, computed from BirthDate. Never stored.
Property Age As %Integer [ Calculated ];

Method AgeGet() As %Integer
{
    if ..BirthDate = "" {
        quit ""
    }
    quit $ZDATE($HOROLOG, 4) - $ZDATE(..BirthDate, 4)
}
```

Explicando:

- `Property Age As %Integer [ Calculated ]` — declara que existe uma propriedade `Age`, mas sem lugar para guardá-la.
- `Method AgeGet() As %Integer` — o nome é **`Age` + `Get`**, sem espaço, exatamente assim. É essa convenção de nome que faz o IRIS entender que esse método é o leitor da propriedade.
- `..BirthDate` — os dois pontos acessam a propriedade da própria instância.
- `$ZDATE(valor, 4)` — converte uma data para o formato que devolve **só o ano**. Assim a subtração dá a diferença em anos. (É uma aproximação: não considera se o aniversário já passou. Vamos refinar no Capítulo 4.4, quando estudarmos datas a fundo.)
- Se `BirthDate` estiver vazia, devolvemos vazio em vez de calcular besteira.

De fora, `patient.Age` funciona exatamente como qualquer outra propriedade. Quem usa não sabe — nem precisa saber — que ela é calculada. Isso se chama encapsulamento, e é uma das grandes vantagens do modelo de objetos.

### 3.6 Coleções

```objectscript
/// Ordered list of allergy names.
Property Allergies As list Of %String;

/// Reference values keyed by test code.
Property ReferenceValues As array Of %Numeric;
```

Repare: **`list Of` e `array Of` se escrevem com a primeira palavra em minúsculas** e o `Of` com O maiúsculo. É a grafia esperada pelo compilador.

Métodos de uma **lista** (`list Of`):

| Chamada | O que faz |
|---|---|
| `obj.Allergies.Insert("penicillin")` | acrescenta ao final |
| `obj.Allergies.InsertAt("latex", 1)` | insere na posição 1, empurrando o resto |
| `obj.Allergies.GetAt(2)` | devolve o valor da posição 2 |
| `obj.Allergies.SetAt("iodine", 2)` | substitui o valor da posição 2 |
| `obj.Allergies.Count()` | quantos itens tem |
| `obj.Allergies.RemoveAt(1)` | remove a posição 1 e reordena |
| `obj.Allergies.Clear()` | esvazia |
| `obj.Allergies.Find("latex")` | devolve a posição onde está, ou vazio |

Métodos de um **array** (`array Of`):

| Chamada | O que faz |
|---|---|
| `obj.ReferenceValues.SetAt(13.5, "HGB")` | guarda o valor sob a chave `HGB` |
| `obj.ReferenceValues.GetAt("HGB")` | devolve o valor daquela chave |
| `obj.ReferenceValues.Count()` | quantas chaves existem |
| `obj.ReferenceValues.RemoveAt("HGB")` | remove aquela chave |
| `obj.ReferenceValues.GetNext(.key)` | percorre as chaves em ordem |

Cuidado com a **ordem dos argumentos** do `SetAt`: primeiro o **valor**, depois a **chave**. É o contrário do que a intuição sugere, e é pegadinha garantida.

O `GetNext(.key)` merece explicação. O ponto antes de `key` significa "por referência" (você viu isso com `.status` no Capítulo 1). Funciona assim:

```objectscript
set key = ""
for {
    set value = obj.ReferenceValues.GetNext(.key)
    quit:key=""              // sai quando não há mais chaves
    write key, " = ", value, !
}
```

Você começa com a chave vazia; a cada volta, o IRIS **atualiza a sua variável `key`** para a próxima chave e devolve o valor correspondente. Quando acabam as chaves, `key` volta a ser vazia e o laço termina. Aquele `quit:key=""` é um `QUIT` com **pós-condicional** — sai apenas se a condição for verdadeira. Pós-condicionais são o Capítulo 4.5; aqui use como receita.

### 3.7 Objeto embutido (`%SerialObject`)

Primeiro se define a classe serial:

```objectscript
/// Postal address. Embedded inside other objects; never stored on its own.
Class LabStudy.Address Extends %SerialObject
{

Property Street As %String(MAXLEN = 120);

Property City As %String(MAXLEN = 80);

Property PostalCode As %String(MAXLEN = 12);

}
```

Depois, na classe persistente, basta declarar uma propriedade daquele tipo:

```objectscript
Property Address As LabStudy.Address;
```

E o uso é encadeando os pontos:

```
LABSTUDY>SET p.Address.City = "Potirendaba"
```

Duas coisas importantes:

- O IRIS **cria automaticamente** o objeto embutido na primeira vez que você atribui algo nele. Você não precisa fazer `SET p.Address = ##class(LabStudy.Address).%New()` — embora fazer isso também funcione.
- Quando você grava o paciente com `%Save()`, o endereço vai junto, **dentro** da mesma linha da tabela. Não existe uma tabela `LabStudy.Address` com IDs próprios.

### 3.8 Relacionamento

Um relacionamento é declarado nos **dois lados**, e as duas declarações precisam combinar.

Na classe do lado "muitos" (o exame pertence a **um** paciente):

```objectscript
Relationship Patient As LabStudy.Patient [ Cardinality = one, Inverse = Exams ];
```

Na classe do lado "um" (o paciente tem **muitos** exames):

```objectscript
Relationship Exams As LabStudy.Exam [ Cardinality = many, Inverse = Patient ];
```

- **`Relationship`** — palavra fixa, no lugar de `Property`.
- **`Cardinality`** — quantos objetos do outro lado esta ponta enxerga. Os valores são `one`, `many`, `parent` e `children`.
- **`Inverse`** — o **nome da propriedade do outro lado**. É isso que amarra as duas pontas. Se você errar esse nome, a compilação falha.

A dupla `parent`/`children` é uma variante mais forte: significa que os filhos **não existem sem o pai**, e apagar o pai apaga os filhos automaticamente. A dupla `one`/`many` é mais solta: são objetos independentes que se conhecem.

O lado `many` se comporta como uma coleção:

```
LABSTUDY>WRITE patient.Exams.Count()
LABSTUDY>SET firstExam = patient.Exams.GetAt(1)
```

E o lado `one` é uma referência direta:

```
LABSTUDY>WRITE exam.Patient.Name
```

Repare no encadeamento `exam.Patient.Name`: o IRIS vai ao disco, carrega o paciente relacionado e lê o nome dele. Isso acontece de forma transparente e se chama *swizzling* — o objeto relacionado é carregado sozinho no momento em que você o toca.

### 3.9 Índices

```
Index NomeDoIndice On Propriedade [ PalavrasChave ] ;
```

Formas comuns:

```objectscript
/// Simple index on one property.
Index NameIdx On Name;

/// Index on two properties, in this order.
Index CityDateIdx On (City, BirthDate);

/// No two objects may have the same record number.
Index RecordIdx On RecordNumber [ Unique ];

/// The record number IS the object id.
Index RecordKey On RecordNumber [ IdKey ];

/// Good for properties with few distinct values.
Index StatusIdx On Status [ Type = bitmap ];

/// Carries extra data so some queries never touch the main record.
Index NameIdx On Name [ Data = (RecordNumber) ];
```

Palavras-chave de índice, uma a uma:

- **`Unique`** — proíbe valores repetidos. Se você tentar gravar um segundo objeto com o mesmo valor, o `%Save()` devolve erro. **Bônus importante:** um índice único faz o compilador **gerar métodos automáticos** com o nome do índice. Para `Index RecordIdx On RecordNumber [ Unique ]`, você ganha de graça:
  - `RecordIdxOpen("REG-001")` — abre o objeto por aquele valor, devolvendo OREF;
  - `RecordIdxExists("REG-001")` — devolve 1 ou 0;
  - `RecordIdxDelete("REG-001")` — apaga por aquele valor.

  Isso é ouro: você passa a poder buscar pelo número do registro sem saber o ID interno.

- **`IdKey`** — vai além do `Unique`: aquela propriedade **passa a ser** o identificador do objeto. `%Id()` devolverá o número de registro em vez de 1, 2, 3. Consequências que a prova adora: a propriedade fica, na prática, imutável depois de gravada, e ela precisa ter valor sempre.

- **`PrimaryKey`** — marca o índice como chave primária **do ponto de vista do SQL**. Sozinho, ele não muda o ID do objeto. `IdKey` e `PrimaryKey` aparecem juntos com frequência, mas **não são a mesma coisa**: `IdKey` afeta o mundo dos objetos; `PrimaryKey` afeta o mundo do SQL.

- **`Type = bitmap`** — um formato de índice compacto e muito rápido para propriedades com **poucos valores distintos** (sexo, status, sim/não) em tabelas grandes. Ele depende de os IDs serem inteiros positivos, o que é o padrão. Para uma propriedade com milhões de valores diferentes (como um nome completo), o bitmap é a escolha errada.

- **`Data = (Prop1, Prop2)`** — guarda cópias dessas propriedades **dentro do próprio índice**. Se uma consulta pedir só essas colunas, o IRIS responde lendo apenas o fichário, sem abrir as fichas. Custa espaço, entrega velocidade.

- **`Extent`** — um índice especial que registra quais IDs existem naquela classe. Útil para contagens rápidas. Costuma aparecer como `[ Extent, Type = bitextent ]`.

### 3.10 Reconstruindo índices

Se você criar um índice numa classe **que já tem dados**, o fichário nasce vazio: os objetos antigos não estão nele. Consultas podem devolver resultados incompletos.

A correção:

```
LABSTUDY>DO ##class(LabStudy.Patient).%BuildIndices()
```

Isso percorre todos os objetos existentes e monta os cartões do fichário. Em tabelas grandes isso demora e deve ser planejado; em estudo, é instantâneo.

Você pode reconstruir apenas alguns índices passando a lista deles: `%BuildIndices($LISTBUILD("NameIdx"))`. A construção de listas com `$LISTBUILD` é o Capítulo 4.2. Existem também parâmetros para controlar travamento e reconstrução em segundo plano — **verificar na documentação oficial** quando precisar disso em produção.

### 3.11 Validando sem gravar

```
LABSTUDY>SET sc = patient.%ValidateObject()
LABSTUDY>IF $$$ISERR(sc) { DO $SYSTEM.Status.DisplayError(sc) }
```

`%ValidateObject()` aplica **todas** as regras declaradas (obrigatoriedade, `MAXLEN`, `VALUELIST`, `MINVAL`...) e devolve um `%Status`, **sem** gravar nada. É perfeito para conferir um objeto antes de decidir se ele vai para o disco.

E aqui está a boa notícia: **o `%Save()` já chama a validação sozinho**. Você não precisa chamar `%ValidateObject()` antes de cada `%Save()`. Chame quando quiser saber o resultado da validação **sem** o efeito de gravar.

---

## 4. Exemplo comentado

Uma classe que usa quase tudo o que vimos:

Arquivo `src/LabStudy/Demo/Person.cls`:

```objectscript
/// Demonstrates property types, keywords, indexes and collections.
Class LabStudy.Demo.Person Extends %Persistent
{

/// Full name. Required, up to 100 characters.
Property Name As %String(MAXLEN = 100) [ Required ];

/// Date of birth. Stored as an internal day number.
Property BirthDate As %Date;

/// Only "M" or "F" are accepted; the screen shows the full word.
Property Sex As %String(VALUELIST = ",M,F", DISPLAYLIST = ",Male,Female");

/// Defaults to true for every new person.
Property Active As %Boolean [ InitialExpression = 1 ];

/// Filled automatically when the object is created.
Property CreatedOn As %TimeStamp [ InitialExpression = {$ZDATETIME($HOROLOG, 3)} ];

/// Computed on demand. Occupies no space at all.
Property Age As %Integer [ Calculated ];

/// Ordered list of free text tags.
Property Tags As list Of %String;

/// Lookup by name.
Index NameIdx On Name;

/// Bitmap index: Sex has only two distinct values.
Index SexIdx On Sex [ Type = bitmap ];

/// Returns the age in whole years, or empty when there is no birth date.
Method AgeGet() As %Integer
{
    if ..BirthDate = "" {
        quit ""
    }
    quit $ZDATE($HOROLOG, 4) - $ZDATE(..BirthDate, 4)
}

}
```

Comentando as decisões:

- `Name` é `Required` e tem `MAXLEN = 100` porque 50 (o padrão) é pouco para nomes completos brasileiros.
- `BirthDate` é `%Date`: guardaremos um número, converteremos na entrada e na saída.
- `Sex` usa `VALUELIST` para restringir e `DISPLAYLIST` para exibir bonito. A vírgula inicial em ambos declara que a vírgula é o separador.
- `Active` nasce ligado, sem ninguém precisar lembrar de ligar.
- `CreatedOn` usa `InitialExpression` com chaves porque o valor precisa ser **calculado no momento da criação**, não um texto fixo. `$ZDATETIME($HOROLOG, 3)` produz o carimbo de data e hora no formato `AAAA-MM-DD HH:MM:SS`, que é exatamente o formato lógico de `%TimeStamp`.
- `Age` é `Calculated` e tem seu `AgeGet()`. Guardar a idade em disco seria um erro de modelagem: ela ficaria errada no dia seguinte ao aniversário. **Idade não é um dado, é um cálculo.**
- `Tags` é uma lista porque a ordem em que as etiquetas foram anotadas pode importar.
- `NameIdx` acelera buscas por nome.
- `SexIdx` é bitmap porque só existem dois valores possíveis — o caso ideal para esse tipo de índice.

### 4.1 Usando no Terminal

```
LABSTUDY>SET p = ##class(LabStudy.Demo.Person).%New()

LABSTUDY>WRITE p.Active, !
1

LABSTUDY>WRITE p.CreatedOn, !
2026-08-19 14:05:11

LABSTUDY>SET p.Name = "Maria Silva"

LABSTUDY>SET p.BirthDate = $ZDATEH("1990-05-17", 3)

LABSTUDY>WRITE p.BirthDate, !
54683

LABSTUDY>WRITE $ZDATE(p.BirthDate, 3), !
1990-05-17

LABSTUDY>SET p.Sex = "F"

LABSTUDY>WRITE p.Age, !
36

LABSTUDY>DO p.Tags.Insert("outpatient")

LABSTUDY>DO p.Tags.Insert("priority")

LABSTUDY>WRITE p.Tags.Count(), " / ", p.Tags.GetAt(2), !
2 / priority

LABSTUDY>SET sc = p.%Save()

LABSTUDY>WRITE $$$ISOK(sc), " id=", p.%Id(), !
1 id=1
```

Pontos a observar:

- `p.Active` e `p.CreatedOn` **já vieram preenchidos** logo após o `%New()`. Esse é o efeito do `InitialExpression`.
- `WRITE p.BirthDate` mostra `54683`, e não a data. É o valor **lógico**. Não é bug; é o desenho do tipo. Para ver a data, converta com `$ZDATE(..., 3)`.
- `p.Age` devolveu um número mesmo sem nunca ter recebido um `SET`. Foi o `AgeGet()` trabalhando. (O valor exato depende da data de hoje.)
- `DO p.Tags.Insert(...)` usa `DO` porque não nos interessa o retorno.

### 4.2 Vendo a validação recusar

```
LABSTUDY>SET bad = ##class(LabStudy.Demo.Person).%New()

LABSTUDY>SET bad.Sex = "X"

LABSTUDY>SET sc = bad.%Save()

LABSTUDY>WRITE $$$ISERR(sc), !
1

LABSTUDY>DO $SYSTEM.Status.DisplayError(sc)
```

O erro exibido aponta dois problemas: `Name` é obrigatório e está vazio, e `Sex` recebeu um valor fora da `VALUELIST`. Repare que **você não escreveu uma linha de validação**: as regras vieram da declaração das propriedades. Esse é o ponto central do capítulo.

---

## 5. Variações e detalhes

### 5.1 Convertendo entre os três rostos

Todo tipo de dado do IRIS oferece métodos de conversão com nomes previsíveis:

```
LABSTUDY>WRITE ##class(%Date).DisplayToLogical("05/17/1990"), !
LABSTUDY>WRITE ##class(%Date).LogicalToDisplay(54683), !
LABSTUDY>WRITE ##class(%Date).LogicalToOdbc(54683), !
1990-05-17
LABSTUDY>WRITE ##class(%Date).OdbcToLogical("1990-05-17"), !
54683
```

O padrão dos nomes é `<Origem>To<Destino>`, com os três rostos sendo `Logical`, `Display` e `Odbc`.

Atenção: os métodos de **exibição** (`Display`) dependem da configuração de idioma e formato da instância. O que sai como `05/17/1990` numa máquina pode sair como `17/05/1990` em outra. Já o formato **ODBC** é sempre `AAAA-MM-DD`. Por isso, **para trocar dados entre sistemas, use sempre ODBC.**

As funções `$ZDATE` e `$ZDATEH` fazem o mesmo trabalho de forma mais direta, e são o que você usará no dia a dia. O código de formato `3` é o formato ODBC. Datas em detalhe: Capítulo 4.4.

### 5.2 Uma propriedade também pode apontar para outro objeto

Além de `Relationship`, existe a forma mais simples: uma propriedade cujo tipo é outra classe persistente.

```objectscript
Property ResponsibleDoctor As LabStudy.Doctor;
```

Isso guarda uma referência ao médico. A diferença para um `Relationship`:

- A propriedade simples é uma via de mão única: o exame conhece o médico, mas o médico **não** tem uma lista automática de exames.
- O `Relationship` é de mão dupla e mantém as duas pontas sincronizadas sozinho.

Use a propriedade simples quando só um lado precisa navegar. Use `Relationship` quando os dois precisam.

### 5.3 O que acontece com a tabela SQL

Já sabemos que a classe persistente vira tabela. Com o que aprendemos agora:

- Cada propriedade simples vira uma **coluna**.
- Um `%SerialObject` embutido vira **várias colunas**, com nomes derivados (por exemplo `Address_City`).
- Uma **coleção** não cabe numa coluna simples: o IRIS projeta uma **tabela filha** separada para ela.
- Uma propriedade `Calculated` **não vira coluna** — a menos que você também a marque como `SqlComputed`, o que a faz aparecer no SQL sendo calculada pelo servidor.
- Um `Relationship` do lado `one` vira uma **coluna com o ID** do objeto relacionado.
- Cada `Index` vira um **índice SQL** que o otimizador de consultas pode usar.

### 5.4 `SqlComputed`: cálculo do lado do SQL

```objectscript
Property FullLabel As %String [ Calculated, SqlComputed,
    SqlComputeCode = { set {*} = {Name}_" ("_{RecordNumber}_")" },
    SqlComputeOnChange = (Name, RecordNumber) ];
```

- **`SqlComputed`** — faz a propriedade ser calculada também quando o acesso vem por SQL.
- **`SqlComputeCode`** — o código do cálculo, entre chaves. Dentro dele:
  - `{*}` significa **"a própria propriedade sendo calculada"**;
  - `{Name}` significa **"o valor da propriedade Name"**.
  Essas chaves com nome dentro são uma sintaxe exclusiva desse contexto.
- **`SqlComputeOnChange`** — quando recalcular. Se a propriedade **não** for `Calculated`, o valor é gravado e recalculado apenas quando as propriedades listadas mudarem.

A combinação decide o comportamento: com `Calculated`, o valor nunca é gravado e é recalculado a cada leitura; sem `Calculated`, o valor é gravado e recalculado nos eventos de `SqlComputeOnChange`.

### 5.5 Adicionar propriedade a uma classe que já tem dados

Isso é comum e não dói: acrescente a propriedade, compile, e os objetos antigos simplesmente terão aquele campo vazio. O IRIS não reescreve os dados existentes.

Dois cuidados:

- Se a nova propriedade for `Required`, os objetos antigos **continuam gravados**, mas qualquer tentativa de regravá-los vai falhar até que o campo seja preenchido.
- Se você acrescentar um índice, rode `%BuildIndices()` para os dados antigos entrarem no fichário.

A evolução de esquema em profundidade é o Capítulo 3.4.

### 5.6 Uma pergunta frequente: quantos índices criar?

Índice não é de graça. Cada índice:

- ocupa espaço em disco;
- torna cada `%Save()` e cada `%DeleteId()` um pouco mais lentos, porque o fichário precisa ser atualizado.

A regra sensata: **indexe o que você realmente busca e filtra**, não tudo. Uma propriedade que nunca aparece num filtro ou numa ordenação não precisa de índice.

---

## 6. Pegadinhas e erros comuns

**1) Esquecer que `%String` tem `MAXLEN` padrão de 50.**
Sintoma: `%Save()` falha ao gravar um texto um pouco maior. Solução: declare `MAXLEN` explicitamente sempre que o campo puder passar de 50.

**2) Confundir parênteses com colchetes.**
Parênteses = parâmetros **do tipo** (`MAXLEN`, `VALUELIST`). Colchetes = palavras-chave **da propriedade** (`Required`, `Calculated`). Trocar os dois não compila.

**3) Esquecer o separador inicial no `VALUELIST`.**
`VALUELIST = "M,F"` está **errado**: o primeiro caractere é o separador, então o separador viraria a letra `M`. O correto é `VALUELIST = ",M,F"`.

**4) Achar que `WRITE p.BirthDate` mostra a data.**
Mostra o valor **lógico** — um número. Converta com `$ZDATE(valor, 3)` ou com `LogicalToOdbc`.

**5) Trocar a ordem dos argumentos de `SetAt`.**
É `SetAt(valor, chave)`, nessa ordem. Inverter guarda a chave como valor e vice-versa, sem dar erro nenhum na hora — o que é pior do que um erro.

**6) Guardar idade em disco.**
Idade muda sozinha com o tempo. Guarde a **data de nascimento** e calcule a idade. Propriedade `Calculated` existe para isso.

**7) Criar índice numa classe com dados e não reconstruir.**
Sintoma: consultas devolvem menos linhas do que deveriam, e você jura que os dados estão lá. Solução: `%BuildIndices()`.

**8) Usar bitmap em propriedade de alta variedade.**
Bitmap é para poucos valores distintos. Num campo de nome completo, ele consome muito espaço e não entrega ganho.

**9) Confundir `IdKey` com `Unique`.**
`Unique` proíbe repetição. `IdKey` faz a propriedade **ser** o identificador do objeto, mudando o que `%Id()` devolve e tornando o valor praticamente imutável depois de gravado.

**10) Confundir `IdKey` com `PrimaryKey`.**
`IdKey` age no mundo dos objetos (o ID). `PrimaryKey` age no mundo do SQL (a chave primária declarada). Podem coexistir; não são sinônimos.

**11) Errar o `Inverse` no relacionamento.**
O `Inverse` tem que conter o **nome da propriedade do outro lado**, não o nome da classe. Errar isso quebra a compilação.

**12) Confundir `Calculated` com `Transient`.**
`Calculated`: sem espaço nenhum, recalculada a cada leitura. `Transient`: tem espaço em memória, some ao gravar. Ambas não vão para o disco, mas por motivos diferentes.

**13) Escrever `List Of` com L maiúsculo.**
A grafia é `list Of` e `array Of`.

**14) Esquecer o ponto e vírgula no fim da propriedade ou do índice.**
`Property X As %String` e `Index Y On X` precisam de `;`. Métodos, não.

---

## 7. MÃO NA MASSA

> Receita de sempre: IRIS ligado → Terminal → `ZN "LABSTUDY"` → escrever a classe no VS Code → `Ctrl+S` → executar.

---

### Exercício 2.1 — Fazendo a validação falhar de propósito

**a) Enunciado:** Crie a classe `LabStudy.Demo.Product`, persistente, com:
- `Code As %String(MAXLEN = 10)`, obrigatória;
- `Description As %String(MAXLEN = 200)`;
- `Category As %String(VALUELIST = ",A,B,C")`;
- `Price As %Numeric(MINVAL = 0, SCALE = 2)`.

Depois, no Terminal, provoque **quatro** falhas de validação diferentes, uma de cada regra, e leia a mensagem de erro de cada uma.

**b) Dica:** Use `%ValidateObject()` para testar sem sujar o banco. Para um texto grande sem digitar 300 letras, use `$JUSTIFY("", 300)`, que produz uma string de 300 espaços — ou `$TRANSLATE($JUSTIFY("",300)," ","x")` para 300 letras `x`.

**c) Como testar:** Cada tentativa deve devolver `$$$ISERR` igual a `1`, e `DisplayError` deve nomear a propriedade culpada.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

`src/LabStudy/Demo/Product.cls`:

```objectscript
/// Used to explore declarative validation rules.
Class LabStudy.Demo.Product Extends %Persistent
{

Property Code As %String(MAXLEN = 10) [ Required ];

Property Description As %String(MAXLEN = 200);

Property Category As %String(VALUELIST = ",A,B,C");

Property Price As %Numeric(MINVAL = 0, SCALE = 2);

}
```

```
LABSTUDY>SET p = ##class(LabStudy.Demo.Product).%New()

LABSTUDY>DO $SYSTEM.Status.DisplayError(p.%ValidateObject())
ERROR #5659: Property 'LabStudy.Demo.Product::Code' required
   (a mensagem exata pode variar conforme a versão)

LABSTUDY>SET p.Code = "ABCDEFGHIJKLM"

LABSTUDY>DO $SYSTEM.Status.DisplayError(p.%ValidateObject())
ERROR #7207: Datatype value 'ABCDEFGHIJKLM' length longer than MAXLEN allowed of 10

LABSTUDY>SET p.Code = "P-001"

LABSTUDY>SET p.Category = "Z"

LABSTUDY>DO $SYSTEM.Status.DisplayError(p.%ValidateObject())
ERROR #7205: Datatype value 'Z' not in VALUELIST ",A,B,C"

LABSTUDY>SET p.Category = "A"

LABSTUDY>SET p.Price = -10

LABSTUDY>DO $SYSTEM.Status.DisplayError(p.%ValidateObject())
ERROR #7203: Datatype value '-10' less than MINVAL allowed of 0

LABSTUDY>SET p.Price = 19.999

LABSTUDY>WRITE p.%ValidateObject(), !
1

LABSTUDY>SET sc = p.%Save()

LABSTUDY>WRITE p.Price, !
20
```

**Por que cada decisão:**

- Usar `%ValidateObject()` em vez de `%Save()` deixa o experimento limpo: nada é gravado enquanto você erra de propósito.
- Passar o `%Status` **direto** como argumento de `DisplayError` (`DisplayError(p.%ValidateObject())`) é um atalho legítimo e enxuto: o resultado da chamada interna vira o argumento da externa.
- O último bloco mostra um efeito colateral que surpreende: `SCALE = 2` **arredondou** `19.999` para `20`. Não houve erro. Parâmetros de tipo nem sempre recusam — às vezes eles **ajustam**. Saber quais recusam e quais ajustam é conhecimento de prova: `MAXLEN` recusa (a menos que `TRUNCATE = 1`), `SCALE` arredonda.
- Os números dos erros (`#5659`, `#7207`...) variam entre versões. Não decore números; **decore o padrão da mensagem**, que sempre nomeia a propriedade e a regra violada.

---

### Exercício 2.2 — Valor inicial e propriedade calculada

**a) Enunciado:** Crie `LabStudy.Demo.Employee` com:
- `Name As %String(MAXLEN = 100)`, obrigatória;
- `HireDate As %Date`, com valor inicial igual à **data de hoje**;
- `Active As %Boolean`, com valor inicial `1`;
- `YearsOfService As %Integer`, **calculada**, devolvendo quantos anos completos se passaram desde a contratação.

Prove, no Terminal, que `HireDate` e `Active` já vêm preenchidos após o `%New()`, e que `YearsOfService` responde sem nunca ter recebido um valor.

**b) Dica:** A data de hoje no formato lógico é simplesmente a primeira parte de `$HOROLOG`. Use `+$HOROLOG` — o sinal de mais força a leitura numérica e devolve só o número de dias. Para o nome do método, lembre da convenção: propriedade + `Get`.

**c) Como testar:** Logo após o `%New()`, `WRITE emp.HireDate` deve mostrar um número grande, e `$ZDATE` desse número deve mostrar a data de hoje. `YearsOfService` deve devolver `0` para um funcionário contratado hoje.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

`src/LabStudy/Demo/Employee.cls`:

```objectscript
/// Demonstrates InitialExpression and calculated properties.
Class LabStudy.Demo.Employee Extends %Persistent
{

Property Name As %String(MAXLEN = 100) [ Required ];

/// Defaults to the current date.
Property HireDate As %Date [ InitialExpression = {+$HOROLOG} ];

/// New employees are active by default.
Property Active As %Boolean [ InitialExpression = 1 ];

/// Whole years since HireDate. Never stored.
Property YearsOfService As %Integer [ Calculated ];

/// Getter for the calculated property YearsOfService.
Method YearsOfServiceGet() As %Integer
{
    if ..HireDate = "" {
        quit ""
    }
    quit $ZDATE($HOROLOG, 4) - $ZDATE(..HireDate, 4)
}

}
```

```
LABSTUDY>SET e = ##class(LabStudy.Demo.Employee).%New()

LABSTUDY>WRITE e.HireDate, !
67352

LABSTUDY>WRITE $ZDATE(e.HireDate, 3), !
2026-08-19

LABSTUDY>WRITE e.Active, !
1

LABSTUDY>WRITE e.YearsOfService, !
0

LABSTUDY>SET e.HireDate = $ZDATEH("2019-03-01", 3)

LABSTUDY>WRITE e.YearsOfService, !
7

LABSTUDY>SET e.YearsOfService = 99

LABSTUDY>WRITE e.YearsOfService, !
7
```

**Por que cada decisão:**

- `InitialExpression = {+$HOROLOG}` — as chaves marcam código a ser executado. `$HOROLOG` devolve algo como `67352,50711` (dias e segundos); o `+` na frente força a interpretação numérica e fica só com a parte dos dias, que é exatamente o formato lógico de `%Date`. Elegante e idiomático.
- O método se chama `YearsOfServiceGet` — nome da propriedade colado com `Get`. Se você escrevesse `GetYearsOfService`, o IRIS **não** reconheceria: seria só um método comum, e a propriedade continuaria devolvendo vazio. Erro clássico.
- As duas últimas linhas mostram algo importante: atribuir a uma propriedade `Calculated` sem método `Set` **não tem efeito**. A leitura continua vindo do `Get`. Se você quisesse permitir escrita, precisaria escrever um `YearsOfServiceSet(value)` — mas aqui não faz sentido, porque tempo de casa é consequência, não escolha.

---

### Exercício 2.3 — Coleções

**a) Enunciado:** Crie `LabStudy.Demo.Basket` com:
- `Owner As %String(MAXLEN = 50)`;
- `Items As list Of %String` — os itens, em ordem;
- `Quantities As array Of %Integer` — a quantidade de cada item, indexada pelo nome do item.

No Terminal: insira três itens na lista, defina quantidades para dois deles, imprima a lista em ordem numerada, percorra o array imprimindo chave e valor, remova o primeiro item da lista e mostre o resultado. Grave e reabra, provando que tudo persistiu.

**b) Dica:** Para percorrer o array, use o padrão do `GetNext(.key)` mostrado na seção 3.6. Para a lista, um `FOR i = 1:1:obj.Items.Count()` resolve.

**c) Como testar:** Depois de reabrir pelo ID, a contagem e os valores devem ser os mesmos.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

`src/LabStudy/Demo/Basket.cls`:

```objectscript
/// Demonstrates list and array collections.
Class LabStudy.Demo.Basket Extends %Persistent
{

Property Owner As %String(MAXLEN = 50);

/// Ordered items.
Property Items As list Of %String;

/// Quantity per item name.
Property Quantities As array Of %Integer;

/// Prints the whole basket.
Method Print() As %Status
{
    write "Basket of ", ..Owner, !
    write "-- items --", !
    for i = 1:1:..Items.Count() {
        write i, ": ", ..Items.GetAt(i), !
    }

    write "-- quantities --", !
    set key = ""
    for {
        set qty = ..Quantities.GetNext(.key)
        quit:key=""
        write key, " = ", qty, !
    }
    quit $$$OK
}

}
```

```
LABSTUDY>SET b = ##class(LabStudy.Demo.Basket).%New()
LABSTUDY>SET b.Owner = "Aziz"

LABSTUDY>DO b.Items.Insert("gloves")
LABSTUDY>DO b.Items.Insert("tubes")
LABSTUDY>DO b.Items.Insert("labels")

LABSTUDY>DO b.Quantities.SetAt(100, "gloves")
LABSTUDY>DO b.Quantities.SetAt(50, "tubes")

LABSTUDY>DO b.Print()
Basket of Aziz
-- items --
1: gloves
2: tubes
3: labels
-- quantities --
gloves = 100
tubes = 50

LABSTUDY>DO b.Items.RemoveAt(1)

LABSTUDY>WRITE b.Items.Count(), " / ", b.Items.GetAt(1), !
2 / tubes

LABSTUDY>SET sc = b.%Save()
LABSTUDY>SET id = b.%Id()
LABSTUDY>SET b = ""

LABSTUDY>SET b = ##class(LabStudy.Demo.Basket).%OpenId(id)
LABSTUDY>DO b.Print()
Basket of Aziz
-- items --
1: tubes
2: labels
-- quantities --
gloves = 100
tubes = 50
```

**Por que cada decisão:**

- O método `Print()` é um `Method`, não um `ClassMethod`, porque ele age **sobre este cesto**. Dentro dele usamos `..Items` e `..Owner`, que só fazem sentido havendo um objeto.
- `RemoveAt(1)` na lista **reordena**: o que era 2 virou 1. Numa lista, as posições são recalculadas. Num array isso não acontece: remover a chave `"gloves"` não mexe nas outras chaves.
- Repare na saída depois do `RemoveAt`: o item `gloves` sumiu da lista, mas a quantidade `gloves = 100` **continuou no array**. A lista e o array são independentes; nada os sincroniza automaticamente. Isso ilustra por que duas coleções paralelas costumam ser uma modelagem frágil — o jeito robusto seria uma classe `Item` com nome e quantidade juntos. Fica a lição de modelagem.
- A saída do array veio em ordem alfabética (`gloves`, depois `tubes`), não na ordem de inserção. É assim que arrays funcionam.

---

### Exercício 2.4 — Índices e os métodos que eles geram

**a) Enunciado:** Crie `LabStudy.Demo.Device` com `Serial As %String(MAXLEN = 30)` e `Model As %String(MAXLEN = 50)`. Declare um índice **único** sobre `Serial`, chamado `SerialIdx`, e um índice comum sobre `Model`.

No Terminal:
1. Grave dois aparelhos com números de série diferentes.
2. Tente gravar um terceiro repetindo um número de série e leia o erro.
3. Use os **métodos gerados** pelo índice único para abrir e para testar existência por número de série, **sem usar o ID**.
4. Acrescente **depois** um índice sobre `Model`, compile e rode `%BuildIndices()`.

**b) Dica:** Os métodos gerados se chamam `<NomeDoIndice>Open`, `<NomeDoIndice>Exists` e `<NomeDoIndice>Delete`.

**c) Como testar:** A terceira gravação deve falhar. `SerialIdxOpen("SN-002")` deve devolver uma OREF válida.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

`src/LabStudy/Demo/Device.cls`:

```objectscript
/// Demonstrates unique indexes and their generated methods.
Class LabStudy.Demo.Device Extends %Persistent
{

Property Serial As %String(MAXLEN = 30) [ Required ];

Property Model As %String(MAXLEN = 50);

/// No two devices may share a serial number.
Index SerialIdx On Serial [ Unique ];

/// Lookup by model.
Index ModelIdx On Model;

}
```

```
LABSTUDY>SET d1 = ##class(LabStudy.Demo.Device).%New()
LABSTUDY>SET d1.Serial = "SN-001", d1.Model = "Analyzer X"
LABSTUDY>WRITE $$$ISOK(d1.%Save()), !
1

LABSTUDY>SET d2 = ##class(LabStudy.Demo.Device).%New()
LABSTUDY>SET d2.Serial = "SN-002", d2.Model = "Analyzer X"
LABSTUDY>WRITE $$$ISOK(d2.%Save()), !
1

LABSTUDY>SET d3 = ##class(LabStudy.Demo.Device).%New()
LABSTUDY>SET d3.Serial = "SN-001", d3.Model = "Analyzer Y"
LABSTUDY>DO $SYSTEM.Status.DisplayError(d3.%Save())
ERROR #5808: Key not unique: LabStudy.Demo.Device:SerialIdx:^...("SN-001")

LABSTUDY>WRITE ##class(LabStudy.Demo.Device).SerialIdxExists("SN-002"), !
1

LABSTUDY>WRITE ##class(LabStudy.Demo.Device).SerialIdxExists("SN-999"), !
0

LABSTUDY>SET found = ##class(LabStudy.Demo.Device).SerialIdxOpen("SN-002")

LABSTUDY>WRITE found.Model, " (id ", found.%Id(), ")", !
Analyzer X (id 2)

LABSTUDY>DO ##class(LabStudy.Demo.Device).%BuildIndices()

LABSTUDY>WRITE "indices rebuilt", !
indices rebuilt
```

**Por que cada decisão:**

- `SET d1.Serial = "SN-001", d1.Model = "Analyzer X"` — um único `SET` pode fazer várias atribuições separadas por vírgula. É idiomático no ObjectScript e você vai ver muito.
- O erro `#5808 Key not unique` é a assinatura do índice único em ação. Repare que a violação só aparece **no `%Save()`**: até ali, o objeto estava perfeitamente montado na memória.
- Os métodos `SerialIdxExists` e `SerialIdxOpen` **não foram escritos por ninguém**. O compilador os gerou porque o índice é `Unique`. Esse é um dos recursos mais úteis e menos conhecidos do IRIS: ele transforma uma chave de negócio (o número de série) numa forma de acesso de primeira classe, sem você precisar do ID interno.
- `%BuildIndices()` foi rodado por disciplina. Neste caso o índice já nasceu junto com os dados, mas o hábito de reconstruir depois de mexer em índices evita um dia ruim.

---

### Exercício 2.5 — PROJETO CONTÍNUO: o paciente ganha corpo

**a) Enunciado:** Evolua o projeto do laboratório:

1. Crie `LabStudy.Address` como `%SerialObject`, com `Street`, `City`, `State` e `PostalCode`.
2. Reescreva `LabStudy.Patient` com:
   - `Name As %String(MAXLEN = 100)`, obrigatória;
   - `BirthDate As %Date`, obrigatória;
   - `RecordNumber As %String(MAXLEN = 20)`, obrigatória;
   - `Sex As %String(VALUELIST = ",M,F", DISPLAYLIST = ",Male,Female")`;
   - `Active As %Boolean [ InitialExpression = 1 ]`;
   - `CreatedOn As %TimeStamp`, preenchida automaticamente na criação;
   - `Address As LabStudy.Address` (embutido);
   - `Allergies As list Of %String`;
   - `Age As %Integer [ Calculated ]`, com seu `Get`;
   - índice **único** `RecordIdx` sobre `RecordNumber`;
   - índice `NameIdx` sobre `Name`;
   - índice **bitmap** `SexIdx` sobre `Sex`.
3. Adapte o `ClassMethod Create` para receber também o sexo, e acrescente um `ClassMethod FindByRecord(recordNumber)` que devolve a OREF do paciente usando o método gerado pelo índice único.
4. Melhore o `Show` para exibir idade, sexo por extenso, cidade e alergias.
5. Crie `LabStudy.Exam`, persistente, com `TestCode As %String(MAXLEN = 20)` obrigatório, `CollectedOn As %TimeStamp`, `ResultValue As %Numeric(SCALE = 3)`, `Unit As %String(MAXLEN = 20)` e um `Relationship Patient` do lado `one`, com inverso `Exams`. Acrescente em `LabStudy.Patient` o `Relationship Exams` do lado `many`.
6. Acrescente a `LabStudy.Exam` um `ClassMethod Register(patientId, testCode, value, unit)` que abre o paciente, cria o exame ligado a ele, grava e devolve o ID.
7. Suba a versão do `LabStudy.App` para `"0.3"` e faça o `Status` mostrar também o total de exames do primeiro paciente.

**b) Dica:** Para exibir o valor de exibição de `Sex` a partir do lógico, use `##class(LabStudy.Patient).SexLogicalToDisplay(valor)` — o compilador gera esse método a partir do par `VALUELIST`/`DISPLAYLIST`. Para o carimbo de criação, `InitialExpression = {$ZDATETIME($HOROLOG, 3)}`.

**c) Como testar:**

```
LABSTUDY>DO ##class(LabStudy.Patient).%KillExtent()
LABSTUDY>DO ##class(LabStudy.Exam).%KillExtent()
LABSTUDY>SET id = ##class(LabStudy.Patient).Create("Maria Silva","1990-05-17","REG-001","F")
LABSTUDY>DO ##class(LabStudy.Exam).Register(id,"HGB",13.5,"g/dL")
LABSTUDY>DO ##class(LabStudy.Exam).Register(id,"GLU",92,"mg/dL")
LABSTUDY>DO ##class(LabStudy.Patient).Show(id)
LABSTUDY>DO ##class(LabStudy.App).Status()
```

O `Show` deve listar os dados do paciente e seus dois exames.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

`src/LabStudy/Address.cls`:

```objectscript
/// Postal address. Always embedded in another object; never stored alone.
Class LabStudy.Address Extends %SerialObject
{

Property Street As %String(MAXLEN = 120);

Property City As %String(MAXLEN = 80);

Property State As %String(MAXLEN = 2);

Property PostalCode As %String(MAXLEN = 12);

}
```

`src/LabStudy/Patient.cls`:

```objectscript
/// A patient of the LabStudy laboratory system.
Class LabStudy.Patient Extends %Persistent [ SqlTableName = PATIENT ]
{

Property Name As %String(MAXLEN = 100) [ Required ];

Property BirthDate As %Date [ Required ];

Property RecordNumber As %String(MAXLEN = 20) [ Required ];

Property Sex As %String(VALUELIST = ",M,F", DISPLAYLIST = ",Male,Female");

Property Active As %Boolean [ InitialExpression = 1 ];

Property CreatedOn As %TimeStamp [ InitialExpression = {$ZDATETIME($HOROLOG, 3)} ];

/// Embedded address. Has no id and no table of its own.
Property Address As LabStudy.Address;

/// Free text allergy names, in the order they were reported.
Property Allergies As list Of %String;

/// Whole years since BirthDate. Never stored.
Property Age As %Integer [ Calculated ];

/// All exams that belong to this patient.
Relationship Exams As LabStudy.Exam [ Cardinality = many, Inverse = Patient ];

/// The laboratory record number is unique across all patients.
Index RecordIdx On RecordNumber [ Unique ];

/// Lookup by name.
Index NameIdx On Name;

/// Sex has only two distinct values: bitmap is the right choice.
Index SexIdx On Sex [ Type = bitmap ];

/// Getter for the calculated property Age.
Method AgeGet() As %Integer
{
    if ..BirthDate = "" {
        quit ""
    }
    quit $ZDATE($HOROLOG, 4) - $ZDATE(..BirthDate, 4)
}

/// Creates and saves a patient. Returns the new id, or "" on failure.
ClassMethod Create(name As %String, birthDate As %String, recordNumber As %String, sex As %String = "") As %String
{
    set patient = ..%New()
    set patient.Name = name
    set patient.BirthDate = $ZDATEH(birthDate, 3)
    set patient.RecordNumber = recordNumber
    set patient.Sex = sex

    set sc = patient.%Save()
    if $$$ISERR(sc) {
        write "Could not save patient:", !
        do $SYSTEM.Status.DisplayError(sc)
        quit ""
    }

    quit patient.%Id()
}

/// Finds a patient by its laboratory record number.
/// Uses the method generated by the unique index RecordIdx.
ClassMethod FindByRecord(recordNumber As %String) As LabStudy.Patient
{
    quit ..RecordIdxOpen(recordNumber)
}

/// Prints the full data of one patient, including exams.
ClassMethod Show(id As %String) As %Status
{
    set patient = ..%OpenId(id)
    if '$ISOBJECT(patient) {
        write "Patient not found: ", id, !
        quit $$$OK
    }

    write "------------------------------", !
    write "Id:      ", patient.%Id(), !
    write "Name:    ", patient.Name, !
    write "Record:  ", patient.RecordNumber, !
    write "Birth:   ", $ZDATE(patient.BirthDate, 3), " (age ", patient.Age, ")", !
    write "Sex:     ", ..SexLogicalToDisplay(patient.Sex), !
    write "City:    ", patient.Address.City, !
    write "Created: ", patient.CreatedOn, !

    write "Allergies (", patient.Allergies.Count(), "):", !
    for i = 1:1:patient.Allergies.Count() {
        write "  - ", patient.Allergies.GetAt(i), !
    }

    write "Exams (", patient.Exams.Count(), "):", !
    for i = 1:1:patient.Exams.Count() {
        set exam = patient.Exams.GetAt(i)
        write "  - ", exam.TestCode, ": ", exam.ResultValue, " ", exam.Unit, !
    }
    write "------------------------------", !
    quit $$$OK
}

}
```

`src/LabStudy/Exam.cls`:

```objectscript
/// A single laboratory test belonging to one patient.
Class LabStudy.Exam Extends %Persistent [ SqlTableName = EXAM ]
{

Property TestCode As %String(MAXLEN = 20) [ Required ];

Property CollectedOn As %TimeStamp [ InitialExpression = {$ZDATETIME($HOROLOG, 3)} ];

Property ResultValue As %Numeric(SCALE = 3);

Property Unit As %String(MAXLEN = 20);

/// The patient this exam belongs to.
Relationship Patient As LabStudy.Patient [ Cardinality = one, Inverse = Exams ];

/// Lookup by test code.
Index TestCodeIdx On TestCode;

/// Creates an exam attached to an existing patient.
/// Returns the new id, or "" on failure.
ClassMethod Register(patientId As %String, testCode As %String, value As %Numeric, unit As %String = "") As %String
{
    set patient = ##class(LabStudy.Patient).%OpenId(patientId)
    if '$ISOBJECT(patient) {
        write "Patient not found: ", patientId, !
        quit ""
    }

    set exam = ..%New()
    set exam.TestCode = testCode
    set exam.ResultValue = value
    set exam.Unit = unit
    set exam.Patient = patient

    set sc = exam.%Save()
    if $$$ISERR(sc) {
        do $SYSTEM.Status.DisplayError(sc)
        quit ""
    }

    quit exam.%Id()
}

}
```

E em `src/LabStudy/App.cls`, atualize o parâmetro e o método `Status`:

```objectscript
Parameter VERSION = "0.3";

/// Naive counters. Will be replaced by SQL queries later.
ClassMethod Status() As %Status
{
    set totalPatients = 0
    for id = 1:1:20 {
        if ##class(LabStudy.Patient).%ExistsId(id) {
            set totalPatients = totalPatients + 1
        }
    }
    write "Patients found (ids 1..20): ", totalPatients, !

    set first = ##class(LabStudy.Patient).%OpenId(1)
    if $ISOBJECT(first) {
        write "Exams of patient 1: ", first.Exams.Count(), !
    }
    quit $$$OK
}
```

Execução esperada:

```
LABSTUDY>SET id = ##class(LabStudy.Patient).Create("Maria Silva","1990-05-17","REG-001","F")

LABSTUDY>SET p = ##class(LabStudy.Patient).%OpenId(id)
LABSTUDY>SET p.Address.City = "Potirendaba"
LABSTUDY>SET p.Address.State = "SP"
LABSTUDY>DO p.Allergies.Insert("penicillin")
LABSTUDY>WRITE $$$ISOK(p.%Save()), !
1

LABSTUDY>DO ##class(LabStudy.Exam).Register(id,"HGB",13.5,"g/dL")
LABSTUDY>DO ##class(LabStudy.Exam).Register(id,"GLU",92,"mg/dL")

LABSTUDY>DO ##class(LabStudy.Patient).Show(id)
------------------------------
Id:      1
Name:    Maria Silva
Record:  REG-001
Birth:   1990-05-17 (age 36)
Sex:     Female
City:    Potirendaba
Created: 2026-08-19 14:22:03
Allergies (1):
  - penicillin
Exams (2):
  - HGB: 13.5 g/dL
  - GLU: 92 mg/dL
------------------------------

LABSTUDY>SET found = ##class(LabStudy.Patient).FindByRecord("REG-001")
LABSTUDY>WRITE found.Name, !
Maria Silva
```

**Por que cada decisão:**

- **`Address` como `%SerialObject`.** Ninguém procura um endereço pelo ID. Ele é um carimbo dentro da ficha. A prova adora exatamente essa escolha de modelagem.
- **Nenhum `%New()` para o endereço.** `SET p.Address.City = "..."` já cria o objeto embutido automaticamente. Se você tivesse feito `%New()` manualmente, também funcionaria; deixar o IRIS cuidar disso é mais limpo.
- **`RecordIdx` único.** O número de registro é a **chave de negócio** do laboratório: dois pacientes não podem ter o mesmo. Declarar o índice único garante isso no nível do banco, e não só na cabeça de quem programou. De brinde, ganhamos `RecordIdxOpen`, usado por `FindByRecord`.
- **`FindByRecord` é uma casca fina sobre o método gerado.** Por que não chamar `RecordIdxOpen` direto no código que usa? Porque `FindByRecord` é um nome que fala a língua do laboratório, e porque, se um dia a busca mudar (por exemplo, passar a ignorar maiúsculas), só um lugar muda.
- **`SexIdx` bitmap.** Duas variações possíveis num universo de milhares de pacientes: é o caso de manual do bitmap.
- **`SexLogicalToDisplay`.** Esse método foi gerado pelo compilador a partir do par `VALUELIST`/`DISPLAYLIST`. Gravamos `F` (uma letra, barata, estável) e mostramos `Female`. Se amanhã o laboratório quiser exibir "Feminino", muda-se **uma linha na definição**, e o banco inteiro continua igual.
- **`Age` calculada, `BirthDate` gravada.** Já discutido, mas vale repetir porque é o tipo de coisa que a prova cobra e a vida real castiga: guarde o fato, calcule a consequência.
- **Relacionamento `one`/`many` em vez de `parent`/`children`.** Um exame é uma entidade com valor próprio — o laboratório quer poder listar "todos os exames de HGB de hoje" sem passar pelos pacientes. Se tivéssemos usado `parent`/`children`, os exames ficariam presos ao paciente e apagar o paciente apagaria os exames. Aqui isso seria arriscado: registro de exame costuma ter valor legal e histórico próprio.
- **`Register` abre o paciente antes de criar o exame.** Atribuir `exam.Patient = patient` exige uma OREF de verdade, não um ID. E o teste com `$ISOBJECT` evita criar um exame órfão apontando para o nada.
- **`Status` ainda é ingênuo.** Continua percorrendo IDs de 1 a 20. Isso é honesto e proposital: no Capítulo 4.6 você vai substituir isso por SQL e sentir a diferença.

---

## 8. Quiz do capítulo

**Q1.** Qual é o comprimento máximo padrão de uma propriedade declarada como `Property Note As %String;`?

- A) Ilimitado.
- B) 50 caracteres.
- C) 255 caracteres.
- D) 32.000 caracteres.

---

**Q2.** Analise:

```objectscript
Property Grade As %String(VALUELIST = "A,B,C");
```

Qual é o problema?

- A) Nenhum; a declaração está correta.
- B) `VALUELIST` só aceita números.
- C) Falta o separador inicial: o primeiro caractere define o delimitador, então o delimitador virou a letra `A`.
- D) `VALUELIST` deve estar entre colchetes, não entre parênteses.

---

**Q3.** Qual é a diferença entre `Calculated` e `Transient`?

- A) São sinônimos.
- B) `Calculated` não ocupa espaço nem em disco nem em memória e é obtida por um método `Get`; `Transient` ocupa espaço em memória mas não é gravada em disco.
- C) `Transient` é recalculada a cada leitura; `Calculated` é gravada.
- D) `Calculated` só existe em classes seriais.

---

**Q4.** Para a propriedade `Property Score As %Integer [ Calculated ];`, qual é o nome correto do método que fornece o valor?

- A) `GetScore()`
- B) `ScoreGet()`
- C) `%GetScore()`
- D) `ScoreCalculate()`

---

**Q5.** Considere `Property Values As array Of %Integer;`. Qual chamada guarda o número 10 sob a chave `"a"`?

- A) `obj.Values.SetAt("a", 10)`
- B) `obj.Values.SetAt(10, "a")`
- C) `obj.Values.Insert("a", 10)`
- D) `obj.Values("a") = 10`

---

**Q6.** Você adicionou um índice novo a uma classe persistente que já contém 10.000 objetos e compilou. As consultas que usam o índice retornam menos linhas do que o esperado. O que fazer?

- A) Recompilar a classe com a flag `b`.
- B) Executar `%BuildIndices()` para popular o índice com os dados existentes.
- C) Executar `%KillExtent()` e recarregar tudo.
- D) Nada; o índice se popula sozinho na próxima consulta.

---

**Q7.** Qual é a diferença entre um índice `Unique` e um índice `IdKey`?

- A) Nenhuma.
- B) `Unique` impede valores repetidos; `IdKey` além disso faz a propriedade ser o identificador do objeto, mudando o que `%Id()` devolve.
- C) `IdKey` permite repetição; `Unique` não.
- D) `IdKey` só vale para SQL; `Unique` só vale para objetos.

---

**Q8.** Para a classe com `Index SerialIdx On Serial [ Unique ]`, quais métodos o compilador gera automaticamente?

- A) Nenhum; índices não geram métodos.
- B) `SerialIdxOpen()`, `SerialIdxExists()` e `SerialIdxDelete()`.
- C) `OpenBySerial()` e `ExistsBySerial()`.
- D) `%OpenSerial()` apenas.

---

**Q9.** Quando um índice `Type = bitmap` é a escolha adequada?

- A) Em propriedades com muitos valores distintos, como nome completo.
- B) Em propriedades com poucos valores distintos, como sexo ou status, em tabelas grandes.
- C) Sempre, pois é o tipo mais rápido em qualquer situação.
- D) Apenas em propriedades de texto longo.

---

**Q10.** `WRITE patient.BirthDate` imprime `54683`. Por quê?

- A) Porque a propriedade foi preenchida incorretamente.
- B) Porque o valor **lógico** de `%Date` é um número inteiro de dias; para ver a data use `$ZDATE(valor, 3)` ou `LogicalToOdbc`.
- C) Porque falta um índice na propriedade.
- D) Porque `%Date` só aceita números.

---

**Q11.** Você quer representar o endereço de um paciente, sabendo que ele nunca será consultado isoladamente nem terá ID próprio. Como declarar?

- A) Uma classe `%Persistent` referenciada por uma propriedade.
- B) Uma classe `%SerialObject` usada como tipo de uma propriedade da classe persistente.
- C) Um `array Of %String`.
- D) Um `Relationship` com `Cardinality = children`.

---

**Q12.** Analise as duas declarações:

```objectscript
// Em LabStudy.Exam
Relationship Patient As LabStudy.Patient [ Cardinality = one, Inverse = Exams ];
// Em LabStudy.Patient
Relationship Exams As LabStudy.Exam [ Cardinality = many, Inverse = Patient ];
```

O que o valor de `Inverse` precisa conter?

- A) O nome da classe do outro lado.
- B) O nome da propriedade de relacionamento do outro lado.
- C) O nome do índice do outro lado.
- D) O valor de `Cardinality` do outro lado.

---

**Q13.** Qual afirmação sobre `%ValidateObject()` está correta?

- A) Ele grava o objeto se a validação passar.
- B) Ele aplica as regras declaradas e devolve `%Status`, sem gravar nada; o `%Save()` já executa essa validação sozinho.
- C) Ele só valida propriedades marcadas como `Required`.
- D) Ele precisa ser chamado obrigatoriamente antes de todo `%Save()`.

---

**Q14.** Uma propriedade declarada como `%Numeric(SCALE = 2)` recebe o valor `19.999`. O que acontece no `%Save()`?

- A) Erro de validação.
- B) O valor é arredondado para `20`.
- C) O valor é truncado para `19`.
- D) O valor é gravado como `19.999`.

---

### Gabarito comentado

**Q1 — Resposta: B.**
- **B está certa:** o `MAXLEN` padrão de `%String` é 50. Textos maiores são recusados na validação.
- **A está errada:** existe limite; é preciso declarar `MAXLEN` para aumentá-lo.
- **C está errada:** 255 é o padrão de outros sistemas, não do IRIS.
- **D está errada:** valores muito grandes exigem `MAXLEN` explícito ou um tipo de stream.

**Q2 — Resposta: C.**
- **C está certa:** em `VALUELIST`, o **primeiro caractere** é o delimitador. Escrevendo `"A,B,C"`, o delimitador vira `A`. O correto é `",A,B,C"`.
- **A está errada:** há problema, e ele é sutil justamente por isso ser cobrado.
- **B está errada:** `VALUELIST` aceita texto normalmente.
- **D está errada:** parâmetros de tipo vão entre parênteses; colchetes são para palavras-chave.

**Q3 — Resposta: B.**
- **B está certa:** `Calculated` não tem armazenamento nenhum e depende de um método `Get`; `Transient` tem espaço em memória mas não vai para o disco.
- **A está errada:** o comportamento é diferente.
- **C está errada:** inverte as definições.
- **D está errada:** `Calculated` funciona em qualquer tipo de classe.

**Q4 — Resposta: B.**
- **B está certa:** a convenção é o nome da propriedade colado com `Get`.
- **A está errada:** `GetScore()` seria apenas um método comum; o IRIS não o associaria à propriedade.
- **C está errada:** o prefixo `%` é reservado ao sistema.
- **D está errada:** não existe essa convenção.

**Q5 — Resposta: B.**
- **B está certa:** a assinatura é `SetAt(valor, chave)` — valor primeiro.
- **A está errada:** inverte os argumentos, guardando `"a"` como valor sob a chave `10`.
- **C está errada:** `Insert` é método de lista, não de array.
- **D está errada:** essa não é a sintaxe de acesso a coleções de objeto.

**Q6 — Resposta: B.**
- **B está certa:** um índice criado depois nasce vazio para os dados já existentes; `%BuildIndices()` os percorre e popula o índice.
- **A está errada:** a flag `b` compila subclasses; não popula índices com dados antigos.
- **C está errada:** apagar tudo é uma solução destrutiva e desnecessária.
- **D está errada:** índices não se populam retroativamente sozinhos.

**Q7 — Resposta: B.**
- **B está certa:** `IdKey` faz a propriedade ser o identificador; `%Id()` passa a devolver esse valor.
- **A está errada:** são coisas diferentes.
- **C está errada:** `IdKey` implica unicidade.
- **D está errada:** `IdKey` afeta o mundo dos objetos; quem é declaradamente SQL é `PrimaryKey`.

**Q8 — Resposta: B.**
- **B está certa:** um índice único gera `<Indice>Open`, `<Indice>Exists` e `<Indice>Delete`.
- **A está errada:** índices únicos geram, sim, métodos.
- **C está errada:** o padrão de nome é o nome do índice como prefixo.
- **D está errada:** o prefixo `%` é do sistema; os métodos gerados usam o nome do índice.

**Q9 — Resposta: B.**
- **B está certa:** bitmap se destina a propriedades de baixa cardinalidade em conjuntos grandes.
- **A está errada:** com muitos valores distintos, o bitmap perde a vantagem e consome espaço.
- **C está errada:** não existe tipo de índice universalmente melhor.
- **D está errada:** texto longo é justamente o caso ruim para bitmap.

**Q10 — Resposta: B.**
- **B está certa:** o formato lógico de `%Date` é um contador inteiro de dias; a exibição exige conversão.
- **A está errada:** o preenchimento está correto; o que muda é o formato interno.
- **C está errada:** índices não têm relação com formato de valor.
- **D está errada:** `%Date` aceita datas; o número é a forma interna dela.

**Q11 — Resposta: B.**
- **B está certa:** `%SerialObject` existe exatamente para dados embutidos, gravados dentro do objeto que os contém.
- **A está errada:** `%Persistent` daria ID e tabela próprios ao endereço.
- **C está errada:** um array de textos perderia a estrutura de campos nomeados.
- **D está errada:** `Cardinality = children` cria objetos persistentes filhos, com identidade própria.

**Q12 — Resposta: B.**
- **B está certa:** `Inverse` aponta para o **nome da propriedade de relacionamento** da outra classe. É isso que amarra as duas pontas.
- **A está errada:** o nome da classe já está no `As`.
- **C está errada:** índices não têm papel nessa ligação.
- **D está errada:** `Cardinality` é declarado separadamente em cada lado.

**Q13 — Resposta: B.**
- **B está certa:** valida e devolve status, sem gravar; e o `%Save()` já faz essa validação internamente.
- **A está errada:** validar não grava.
- **C está errada:** valida todas as regras declaradas, incluindo `MAXLEN`, `VALUELIST` e `MINVAL`.
- **D está errada:** é opcional, útil quando você quer o resultado sem o efeito de gravar.

**Q14 — Resposta: B.**
- **B está certa:** `SCALE` arredonda para o número de casas indicado. Não é um erro de validação.
- **A está errada:** `SCALE` ajusta em vez de recusar.
- **C está errada:** o comportamento é arredondamento, não truncamento.
- **D está errada:** o valor não é preservado com três casas quando `SCALE = 2`.

---

## 9. Resumo relâmpago

1. `Property Nome As Tipo(paramsDeTipo) [ palavrasChave ];` — **parênteses** para parâmetros do tipo, **colchetes** para palavras-chave, **ponto e vírgula** no fim.
2. `%String` sem `MAXLEN` aceita **50 caracteres**.
3. `VALUELIST` e `DISPLAYLIST` começam com o **caractere separador**: `",M,F"`, não `"M,F"`.
4. Todo valor tem três rostos: **lógico** (o que fica na propriedade), **exibição** (depende do idioma) e **ODBC** (`AAAA-MM-DD`, universal).
5. `%Date` lógico = número de dias. `%Time` lógico = segundos desde a meia-noite. `%TimeStamp` lógico = `AAAA-MM-DD HH:MM:SS`. `%Boolean` lógico = 1/0.
6. `Required` obriga; `InitialExpression` preenche no `%New()`; código em `InitialExpression` vai **entre chaves**.
7. `Calculated` = sem espaço nenhum, valor vem de `<Prop>Get()`. `Transient` = espaço em memória, nada em disco.
8. O método leitor se chama **`<Propriedade>Get()`** e o escritor **`<Propriedade>Set(value)`** — nome colado, nessa ordem.
9. `list Of` mantém **ordem** (`Insert`, `GetAt(i)`); `array Of` usa **chaves** (`SetAt(valor, chave)` — valor primeiro).
10. `%SerialObject` é embutido: sem ID, sem tabela própria, gravado dentro do objeto que o contém.
11. `Relationship` precisa dos dois lados combinando: `Cardinality` (`one`/`many` ou `parent`/`children`) e `Inverse` com o **nome da propriedade** do outro lado.
12. Índice acelera leitura, **custa espaço e torna a escrita um pouco mais lenta**. Indexe o que você filtra.
13. `Unique` proíbe repetição e **gera** `<Indice>Open`, `<Indice>Exists` e `<Indice>Delete`.
14. `IdKey` faz a propriedade **ser o ID**; `PrimaryKey` declara a chave primária **do SQL**. Não são a mesma coisa.
15. `Type = bitmap` é para **poucos valores distintos** em tabelas grandes.
16. Índice criado depois dos dados exige **`%BuildIndices()`**.
17. `%ValidateObject()` valida sem gravar; o `%Save()` já valida sozinho.
18. `MAXLEN` **recusa** (salvo `TRUNCATE = 1`); `SCALE` **arredonda**. Nem todo parâmetro de tipo dá erro.

---

## 10. Cartões de memorização

**Frente:** Qual o `MAXLEN` padrão de `%String`?
**Verso:** 50 caracteres.

**Frente:** Como se escreve corretamente um `VALUELIST` com os valores M e F?
**Verso:** `VALUELIST = ",M,F"` — o primeiro caractere é o separador.

**Frente:** O que fica guardado numa propriedade `%Date`?
**Verso:** O valor lógico: um número inteiro de dias contados de uma data-base interna.

**Frente:** Qual formato usar para trocar datas entre sistemas?
**Verso:** ODBC (`AAAA-MM-DD`), porque não depende de idioma nem de configuração local.

**Frente:** Qual o formato lógico de `%TimeStamp`?
**Verso:** O texto `AAAA-MM-DD HH:MM:SS`.

**Frente:** Diferença entre `Calculated` e `Transient`.
**Verso:** `Calculated`: sem espaço algum, valor vem do método `Get`. `Transient`: tem espaço em memória, mas não é gravada em disco.

**Frente:** Como se chama o método que fornece o valor da propriedade `Age`?
**Verso:** `AgeGet()` — nome da propriedade colado com `Get`.

**Frente:** Como declarar um valor padrão calculado na criação do objeto?
**Verso:** `[ InitialExpression = {expressão} ]` — o código vai entre chaves.

**Frente:** Assinatura do `SetAt` de um `array Of`.
**Verso:** `SetAt(valor, chave)` — o valor vem primeiro.

**Frente:** Como percorrer todas as chaves de um `array Of`?
**Verso:** `set key="" for { set v = obj.Coll.GetNext(.key) quit:key="" ... }`.

**Frente:** Quando usar `%SerialObject`?
**Verso:** Para dados que só existem embutidos em outro objeto, sem ID nem tabela próprios (endereço, faixa de referência).

**Frente:** O que `Inverse` deve conter num `Relationship`?
**Verso:** O nome da **propriedade de relacionamento** do outro lado.

**Frente:** Diferença entre `one`/`many` e `parent`/`children`.
**Verso:** `one`/`many` liga objetos independentes. `parent`/`children` cria dependência: apagar o pai apaga os filhos.

**Frente:** O que um índice `Unique` gera automaticamente?
**Verso:** Os métodos `<Indice>Open()`, `<Indice>Exists()` e `<Indice>Delete()`.

**Frente:** Diferença entre `IdKey` e `PrimaryKey`.
**Verso:** `IdKey` faz a propriedade ser o identificador do objeto (muda `%Id()`). `PrimaryKey` declara a chave primária no nível do SQL.

**Frente:** Quando usar índice bitmap?
**Verso:** Em propriedades com poucos valores distintos, em tabelas com muitos registros.

**Frente:** Criei um índice numa classe que já tinha dados. E agora?
**Verso:** Rode `DO ##class(Pacote.Classe).%BuildIndices()`.

**Frente:** O que faz `%ValidateObject()`?
**Verso:** Aplica todas as regras declaradas e devolve `%Status`, **sem gravar**. O `%Save()` já valida por conta própria.

**Frente:** `MAXLEN` estourado dá erro ou corta?
**Verso:** Dá erro de validação — a menos que `TRUNCATE = 1`, quando corta silenciosamente.

**Frente:** `SCALE = 2` recebendo `19.999`?
**Verso:** Arredonda para `20`. `SCALE` ajusta, não recusa.

---

Digite CONTINUAR para o próximo capítulo.
