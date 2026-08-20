# Apostila InterSystems ObjectScript Specialist
## Capítulo 4 — T1.4 Uses Complex Structures (JSON e streams)

> Último tópico do domínio **T1 — Manages Data Model**. Aqui você aprende a lidar com dados que não cabem numa propriedade simples: estruturas aninhadas (JSON) e conteúdos grandes demais para uma string (streams).

---

## 1. Objetivo do capítulo

Ao terminar este capítulo, você será capaz de:

1. Explicar o que é **JSON** e por que ele existe, sem depender de conhecimento prévio.
2. Criar objetos e vetores dinâmicos com a **sintaxe literal** `{ }` e `[ ]`.
3. Usar os métodos das classes **`%DynamicObject`** e **`%DynamicArray`**: `%Set`, `%Get`, `%Remove`, `%IsDefined`, `%Size`, `%GetTypeOf`, `%Push`, `%Pop`.
4. Converter entre texto JSON e estrutura com **`%FromJSON()`** e **`%ToJSON()`**.
5. Percorrer estruturas dinâmicas com **`%GetIterator()`** e **`%GetNext()`**.
6. Distinguir **`null` de JSON** de **elemento inexistente**.
7. Usar o **`%JSON.Adaptor`** para exportar e importar objetos persistentes como JSON, com `%JSONExport`, `%JSONExportToString` e `%JSONImport`.
8. Entender o que é um **stream**, por que ele existe, e a diferença entre `%Stream.GlobalCharacter`, `%Stream.GlobalBinary` e `%Stream.FileCharacter`.
9. Escrever e ler streams com `Write`, `WriteLine`, `Read`, `ReadLine`, `Rewind`, `MoveToEnd`, `Clear` e as propriedades `Size` e `AtEnd`.
10. Usar um stream como **propriedade de uma classe persistente**.
11. Evoluir o projeto: laudos em stream, exportação de pacientes em JSON e importação de exames a partir de JSON.

---

## 2. O conceito em linguagem de gente

### 2.1 O problema que o JSON resolve

Volte ao laboratório. Um sistema parceiro precisa mandar para você a ficha de um paciente. Ele não pode mandar o objeto do jeito que ele existe na memória dele — a memória dele é outra máquina, outro programa, talvez outra linguagem.

Então ele precisa **escrever a ficha num pedaço de texto**, de um jeito que qualquer sistema saiba ler de volta.

Poderia ser assim:

```
Maria Silva;1990-05-17;REG-001
```

Funciona, mas é frágil: quem lê precisa saber de cor que o primeiro campo é o nome. E se o paciente tiver duas alergias? E se tiver um endereço com quatro subcampos?

O **JSON** é um formato de texto que resolve isso carregando **os nomes junto com os valores**, e permitindo **estruturas dentro de estruturas**:

```json
{
  "name": "Maria Silva",
  "birthDate": "1990-05-17",
  "recordNumber": "REG-001",
  "address": {
    "city": "Potirendaba",
    "state": "SP"
  },
  "allergies": ["penicillin", "latex"]
}
```

Duas construções bastam para entender tudo:

- **Chaves `{ }`** delimitam um **objeto**: um conjunto de pares `"nome": valor`. É como uma ficha com campos etiquetados.
- **Colchetes `[ ]`** delimitam um **vetor** (*array*): uma sequência ordenada de valores, sem etiquetas. É como uma lista numerada.

E os valores podem ser: texto (entre aspas duplas), número (sem aspas), `true`, `false`, `null`, outro objeto, ou outro vetor. É só isso. O formato inteiro cabe nesse parágrafo.

### 2.2 Estrutura dinâmica: uma ficha que não precisa de formulário

Até agora, toda vez que você quis guardar dados, primeiro criou uma **classe** dizendo quais campos existem. Isso é ótimo quando você conhece o formato de antemão.

Mas às vezes você não conhece. Chegou um JSON de um sistema parceiro e você quer olhar dentro dele **antes** de decidir o que fazer. Criar uma classe para cada formato possível seria inviável.

Para isso o IRIS oferece as **estruturas dinâmicas**:

- **`%DynamicObject`** — a ficha sem formulário: você acrescenta campos na hora, com os nomes que quiser.
- **`%DynamicArray`** — a lista sem formulário.

Analogia: a classe persistente é um formulário impresso na gráfica. A estrutura dinâmica é uma folha em branco onde você escreve os rótulos à mão, na hora. Menos rígida, menos validada, mas infinitamente mais flexível.

Quando usar cada uma? Regra de bolso:

- **Dados que você controla e que precisam durar** → classe persistente, com propriedades declaradas e validação.
- **Dados que estão de passagem** (chegaram por uma API, vão sair para uma API) → estrutura dinâmica.

Na prática, um sistema profissional usa as duas: recebe JSON numa estrutura dinâmica, examina, valida, e então preenche objetos persistentes de verdade.

### 2.3 O problema que o stream resolve

Agora um problema diferente.

Uma propriedade `%String` tem limite de tamanho. Mesmo aumentando o `MAXLEN`, existe um teto imposto pelo próprio IRIS para uma string. Ele é grande — alguns milhões de caracteres —, mas é um teto.

Só que às vezes você precisa guardar coisas maiores: o texto completo de um laudo com anexos, um arquivo enviado por um usuário, uma imagem, um PDF.

O **stream** é a resposta. A palavra significa "fluxo", e a analogia é literal.

Pense na diferença entre um **balde** e uma **mangueira**:

- A **string** é o balde: você pega tudo de uma vez, carrega tudo de uma vez, e se não couber no balde, não dá.
- O **stream** é a mangueira: a água passa aos poucos. Você escreve um pedaço, depois outro, depois outro. E lê da mesma forma: um pedaço por vez, no seu ritmo.

Consequência importantíssima disso: um stream tem uma **posição atual**, como o cabeçote de um toca-fitas. Depois de escrever, o cabeçote está no fim. Se você tentar ler sem antes **rebobinar**, não lê nada. Esse é o erro número um de quem começa com streams, e vamos repetir isso algumas vezes ao longo do capítulo.

### 2.4 Os tipos de stream

Duas perguntas decidem qual classe usar.

**Pergunta 1: é texto ou são bytes?**

- **Texto** (*character*) — um laudo, um XML, um JSON. O IRIS trata conversão de caracteres.
- **Bytes** (*binary*) — uma imagem, um PDF, um arquivo compactado. Nada é interpretado.

**Pergunta 2: guardar dentro do banco ou num arquivo do disco?**

- **Global** — o conteúdo mora dentro do banco de dados do IRIS, junto com os outros dados.
- **File** — o conteúdo mora num arquivo do sistema operacional, e o IRIS guarda apenas o caminho.

Cruzando as duas perguntas:

| | Texto | Bytes |
|---|---|---|
| **No banco** | `%Stream.GlobalCharacter` | `%Stream.GlobalBinary` |
| **Em arquivo** | `%Stream.FileCharacter` | `%Stream.FileBinary` |

Há também `%Stream.TmpCharacter` e `%Stream.TmpBinary`, para conteúdo temporário que não precisa sobreviver à sessão.

Na dúvida, para uma propriedade de uma classe persistente, use **`%Stream.GlobalCharacter`** (texto) ou **`%Stream.GlobalBinary`** (binário): o conteúdo fica no banco, é gravado junto com o objeto e acompanha os backups.

---

## 3. A sintaxe explicada

### 3.1 Criando estruturas dinâmicas com literais

O ObjectScript entende JSON **direto no código**:

```objectscript
set patient = {"name":"Maria Silva","age":36,"active":true}
set numbers = [10, 20, 30]
set empty = {}
set emptyList = []
```

- **`{ ... }`** cria um **`%DynamicObject`**.
- **`[ ... ]`** cria um **`%DynamicArray`**.
- As chaves (os nomes dos campos) vão entre **aspas duplas**.
- Texto vai entre aspas duplas; números, `true`, `false` e `null` vão sem aspas.

Você pode aninhar à vontade:

```objectscript
set patient = {
    "name": "Maria Silva",
    "address": { "city": "Potirendaba", "state": "SP" },
    "allergies": ["penicillin", "latex"]
}
```

E pode inserir valores calculados usando **parênteses**:

```objectscript
set name = "Maria"
set patient = {"name": (name), "createdOn": ($ZDATE($HOROLOG, 3))}
```

Os parênteses dizem ao compilador: *"isto não é um texto literal, é uma expressão para avaliar"*. Sem eles, `"name": name` guardaria a palavra `name`.

### 3.2 Lendo e escrevendo campos

Duas formas, ambas válidas:

**Forma com ponto** — curta e legível:

```objectscript
set patient.email = "maria@example.com"
write patient.name, !
```

**Forma com métodos** — necessária quando o nome do campo está numa variável ou tem caracteres especiais:

```objectscript
do patient.%Set("email", "maria@example.com")
write patient.%Get("name"), !
```

Os métodos principais de `%DynamicObject`:

| Chamada | O que faz |
|---|---|
| `obj.%Set("chave", valor)` | define o valor da chave |
| `obj.%Get("chave")` | devolve o valor, ou vazio se não existir |
| `obj.%Get("chave", "padrao")` | devolve o padrão quando a chave não existe |
| `obj.%Remove("chave")` | apaga a chave |
| `obj.%IsDefined("chave")` | devolve 1 ou 0 |
| `obj.%Size()` | quantos elementos existem |
| `obj.%GetTypeOf("chave")` | devolve o tipo do valor, como texto |
| `obj.%GetIterator()` | devolve um percorredor de todos os elementos |
| `obj.%ToJSON()` | devolve o texto JSON correspondente |

### 3.3 Vetores dinâmicos

```objectscript
set list = [10, 20, 30]

write list.%Get(0), !          // 10  -- ATENÇÃO: começa em ZERO
write list.%Size(), !          // 3

do list.%Push(40)              // acrescenta ao final
write list.%Pop(), !           // 40  -- remove e devolve o último

do list.%Set(1, 99)            // troca a posição 1
```

**O ponto mais importante desta seção:** vetores dinâmicos são indexados **a partir de zero**. O primeiro elemento é `%Get(0)`.

Isso contrasta com as coleções `list Of` do Capítulo 2, que começam em **1**. Ter duas convenções diferentes no mesmo produto é confuso, e a prova explora exatamente essa confusão. Grave:

- **`list Of %String`** (coleção de classe) → começa em **1**, usa `GetAt()`.
- **`%DynamicArray`** (estrutura dinâmica) → começa em **0**, usa `%Get()`.

### 3.4 Convertendo texto JSON em estrutura, e vice-versa

**De texto para estrutura:**

```objectscript
set json = "{""name"":""Maria"",""age"":36}"
set obj = ##class(%DynamicObject).%FromJSON(json)
write obj.name, !
```

Repare nas **aspas duplas dobradas**: dentro de uma string do ObjectScript, para escrever uma aspa dupla literal, você a escreve **duas vezes**. `""name""` produz `"name"`. Isso é sintaxe da linguagem, não do JSON.

`%FromJSON` também aceita um stream ou o caminho de um arquivo como origem, o que é muito útil para JSON grande.

**De estrutura para texto:**

```objectscript
set obj = {"name":"Maria","age":36}
write obj.%ToJSON(), !
```

Saída:

```
{"name":"Maria","age":36}
```

`%ToJSON()` devolve o texto compacto, sem espaços nem quebras de linha. Se você passar um destino como argumento, ele escreve lá em vez de devolver.

### 3.5 Descobrindo o tipo de um valor

```objectscript
set obj = {"a":"text","b":42,"c":true,"d":null,"e":{},"f":[]}

write obj.%GetTypeOf("a"), !    // string
write obj.%GetTypeOf("b"), !    // number
write obj.%GetTypeOf("c"), !    // boolean
write obj.%GetTypeOf("d"), !    // null
write obj.%GetTypeOf("e"), !    // object
write obj.%GetTypeOf("f"), !    // array
write obj.%GetTypeOf("zzz"), !  // unassigned
```

Os valores possíveis são: `string`, `number`, `boolean`, `null`, `object`, `array`, `oref` e `unassigned`.

**O par que mais cai na prova é `null` contra `unassigned`:**

- **`null`** significa *"esse campo existe e o valor dele é explicitamente nulo"*. No JSON de origem estava escrito `"campo": null`.
- **`unassigned`** significa *"esse campo simplesmente não existe"*.

Em muitos protocolos essa diferença é semântica: "o paciente não informou o telefone" (null) é diferente de "esse sistema nem tem campo de telefone" (unassigned).

`%GetTypeOf()` é a forma segura de distinguir os dois, porque `%Get()` devolve vazio nos dois casos.

### 3.6 Percorrendo uma estrutura dinâmica

Você não sabe quais chaves existem? Use um **iterador**:

```objectscript
set iterator = obj.%GetIterator()
while iterator.%GetNext(.key, .value) {
    write key, " = ", value, !
}
```

- **`%GetIterator()`** devolve um objeto que sabe caminhar pelos elementos.
- **`%GetNext(.key, .value)`** avança um passo. Devolve `1` enquanto houver elementos e `0` quando acabar. Repare nos **pontos**: `key` e `value` são parâmetros de saída — exatamente o que você aprendeu no Capítulo 3.
- Num objeto, `key` é o nome do campo. Num vetor, `key` é o índice numérico (começando em zero).
- **`while condição { ... }`** repete enquanto a condição for verdadeira. Estruturas de repetição são o Capítulo 17; aqui use como receita.

Uma versão com três argumentos também existe, trazendo o tipo:

```objectscript
while iterator.%GetNext(.key, .value, .type) {
    write key, " (", type, ") = ", value, !
}
```

### 3.7 O `%JSON.Adaptor`: objetos persistentes viram JSON sozinhos

Converter propriedade por propriedade seria trabalhoso e frágil. O IRIS oferece um atalho: basta **herdar também** de `%JSON.Adaptor`.

```objectscript
Class LabStudy.Demo.Item Extends (%Persistent, %JSON.Adaptor)
{
Property Code As %String;
Property Quantity As %Integer;
}
```

Repare na herança múltipla com parênteses, e em `%Persistent` vindo **primeiro** — a superclasse primária, como você aprendeu no Capítulo 1.

Com isso, a classe ganha:

| Método | O que faz |
|---|---|
| `obj.%JSONExport()` | escreve o JSON do objeto no dispositivo atual (a tela) |
| `obj.%JSONExportToString(.str)` | devolve `%Status` e coloca o JSON na variável `str` |
| `obj.%JSONImport(json)` | preenche o objeto a partir de um texto JSON, devolvendo `%Status` |

```
LABSTUDY>SET i = ##class(LabStudy.Demo.Item).%New()
LABSTUDY>SET i.Code = "GLV", i.Quantity = 100
LABSTUDY>DO i.%JSONExport()
{"Code":"GLV","Quantity":100}

LABSTUDY>SET sc = i.%JSONExportToString(.text)
LABSTUDY>WRITE text, !
{"Code":"GLV","Quantity":100}

LABSTUDY>SET novo = ##class(LabStudy.Demo.Item).%New()
LABSTUDY>SET sc = novo.%JSONImport("{""Code"":""TUB"",""Quantity"":50}")
LABSTUDY>WRITE novo.Code, " ", novo.Quantity, !
TUB 50
```

Para ajustar o formato, existem **parâmetros de propriedade** próprios do adaptador:

```objectscript
Property Code As %String(%JSONFIELDNAME = "code");

Property InternalNote As %String(%JSONINCLUDE = "none");

Property Password As %String(%JSONINCLUDE = "inputonly");
```

- **`%JSONFIELDNAME`** — o nome que o campo terá no JSON. Serve para converter `RecordNumber` em `recordNumber`, por exemplo.
- **`%JSONINCLUDE`** — controla a direção: `"all"` (padrão), `"outputonly"` (só exporta), `"inputonly"` (só importa) ou `"none"` (ignorada nos dois sentidos).
- **`%JSONNULL`** — se `1`, propriedades vazias saem como `null` em vez de serem omitidas.
- **`%JSONIGNORENULL`** — controla como strings vazias e nulos são tratados na importação.

É possível ainda definir **mapeamentos nomeados** num bloco `XData`, permitindo que a mesma classe seja exportada em formatos diferentes conforme o destino. A sintaxe exata desses blocos e o conjunto completo de parâmetros: **verificar na documentação oficial**.

### 3.8 Streams: a sintaxe

**Criando um stream avulso:**

```objectscript
set stream = ##class(%Stream.GlobalCharacter).%New()
```

**Escrevendo:**

```objectscript
do stream.Write("primeira parte ")
do stream.Write("segunda parte")
do stream.WriteLine("")
do stream.WriteLine("uma linha inteira")
```

- **`Write(texto)`** acrescenta o texto na posição atual.
- **`WriteLine(texto)`** acrescenta o texto **mais um terminador de linha**.

**Rebobinando e lendo:**

```objectscript
do stream.Rewind()
while 'stream.AtEnd {
    write stream.Read(20)
}
```

- **`Rewind()`** volta o cabeçote para o início. **Sem isso, a leitura não devolve nada**, porque o cabeçote está no fim depois da escrita.
- **`Read(tamanho)`** lê até aquele número de caracteres a partir da posição atual e avança o cabeçote.
- **`AtEnd`** é uma **propriedade** (não um método) que vale `1` quando o cabeçote chegou ao fim. Note que se escreve `stream.AtEnd`, sem parênteses.
- **`ReadLine()`** lê até o próximo terminador de linha.
- **`MoveToEnd()`** manda o cabeçote para o fim, para você continuar escrevendo sem apagar o que já existe.
- **`Clear()`** esvazia o stream.
- **`Size`** é uma **propriedade** com o tamanho total em caracteres (ou bytes).
- **`CopyFrom(outroStream)`** copia o conteúdo inteiro de outro stream.

Resumindo o ciclo mental: **escreveu → rebobinou → leu.**

**Stream ligado a um arquivo:**

```objectscript
set fileStream = ##class(%Stream.FileCharacter).%New()
set sc = fileStream.LinkToFile("/home/irisowner/report.txt")
do fileStream.WriteLine("linha do relatorio")
set sc = fileStream.%Save()
```

- **`LinkToFile(caminho)`** aponta o stream para um arquivo do sistema operacional.
- **`%Save()`** é o que efetivamente grava o conteúdo no arquivo.

### 3.9 Stream como propriedade de uma classe persistente

```objectscript
Class LabStudy.Demo.Report Extends %Persistent
{
Property Title As %String(MAXLEN = 200);
Property Body As %Stream.GlobalCharacter;
}
```

O uso é direto, e o IRIS cuida do resto:

```
LABSTUDY>SET r = ##class(LabStudy.Demo.Report).%New()
LABSTUDY>SET r.Title = "Daily summary"
LABSTUDY>DO r.Body.WriteLine("Line one")
LABSTUDY>DO r.Body.WriteLine("Line two")
LABSTUDY>SET sc = r.%Save()
```

Três coisas acontecem sozinhas e você precisa saber:

1. O IRIS **cria o stream automaticamente** na primeira vez que você o toca. Não é preciso `%New()` para ele.
2. Quando o objeto que o contém é gravado, **o stream é gravado junto**.
3. Ao reabrir o objeto com `%OpenId()`, o stream vem posicionado **no início**, pronto para leitura. Mesmo assim, chamar `Rewind()` antes de ler é um hábito que nunca faz mal.

---

## 4. Exemplo comentado

Uma classe que combina JSON e streams:

Arquivo `src/LabStudy/Demo/Message.cls`:

```objectscript
/// Demonstrates dynamic objects, the JSON adaptor and character streams.
Class LabStudy.Demo.Message Extends (%Persistent, %JSON.Adaptor)
{

/// Sender name. Exported as "sender" in JSON.
Property Sender As %String(%JSONFIELDNAME = "sender", MAXLEN = 100) [ Required ];

/// Subject line.
Property Subject As %String(%JSONFIELDNAME = "subject", MAXLEN = 200);

/// Internal note: never leaves the system.
Property InternalNote As %String(%JSONINCLUDE = "none", MAXLEN = 200);

/// Long body text. Streams are not projected to JSON by the adaptor.
Property Body As %Stream.GlobalCharacter;

/// Builds a dynamic object summarising this message.
Method ToSummary() As %DynamicObject
{
    set summary = {}
    set summary.sender = ..Sender
    set summary.subject = ..Subject
    set summary.bodySize = ..Body.Size
    set summary.hasNote = $SELECT(..InternalNote '= "": 1, 1: 0)
    quit summary
}

/// Appends a paragraph to the body stream.
Method AddParagraph(text As %String) As %Status
{
    do ..Body.MoveToEnd()
    do ..Body.WriteLine(text)
    quit $$$OK
}

/// Prints the whole body, reading it in small chunks.
Method PrintBody() As %Status
{
    do ..Body.Rewind()
    while '..Body.AtEnd {
        write ..Body.Read(32)
    }
    write !
    quit $$$OK
}

/// Creates a message from a JSON string.
/// Returns the new id, or "" on failure.
ClassMethod FromJSON(json As %String) As %String
{
    set data = ##class(%DynamicObject).%FromJSON(json)

    if 'data.%IsDefined("sender") {
        write "JSON is missing the 'sender' field", !
        quit ""
    }

    set message = ..%New()
    set message.Sender = data.sender
    set message.Subject = data.%Get("subject", "(no subject)")

    if data.%GetTypeOf("lines") = "array" {
        set iterator = data.lines.%GetIterator()
        while iterator.%GetNext(.index, .line) {
            do message.AddParagraph(line)
        }
    }

    set sc = message.%Save()
    if $$$ISERR(sc) {
        do $SYSTEM.Status.DisplayError(sc)
        quit ""
    }

    quit message.%Id()
}

}
```

Comentando as decisões:

- **`Extends (%Persistent, %JSON.Adaptor)`** — herança múltipla, com a classe persistente **primeiro**, como manda a regra da superclasse primária.
- **`%JSONFIELDNAME = "sender"`** — a propriedade se chama `Sender` (estilo do IRIS) mas sai como `sender` no JSON (estilo comum em APIs). Assim, cada mundo mantém a sua convenção.
- **`%JSONINCLUDE = "none"`** em `InternalNote` — essa informação nunca sai nem entra por JSON. É a forma declarativa de proteger um campo interno.
- **`ToSummary()` devolve um `%DynamicObject`** montado à mão. Repare que ele **não** é o objeto inteiro: é um resumo com campos escolhidos, inclusive um campo calculado (`bodySize`) que não é propriedade de nada. Estruturas dinâmicas brilham exatamente nesse tipo de tarefa.
- **`$SELECT(condição: valor, 1: outroValor)`** — devolve o primeiro valor cuja condição for verdadeira. O `1` final significa "senão". É a forma compacta de um "se isto, aquilo; senão, aquilo outro". Detalhes no Capítulo 17.
- **`AddParagraph` chama `MoveToEnd()` antes de escrever.** Se o objeto acabou de ser reaberto do disco, o cabeçote está no início; escrever sem mover sobrescreveria o começo. Hábito defensivo.
- **`PrintBody` chama `Rewind()` antes de ler** e lê em pedaços de 32 caracteres. O tamanho pequeno é didático: mostra que o stream é lido aos poucos. Em código real você usaria pedaços maiores.
- **`FromJSON` valida antes de usar.** `%IsDefined("sender")` garante que o campo obrigatório veio. `%Get("subject", "(no subject)")` fornece um padrão quando o campo é opcional. `%GetTypeOf("lines") = "array"` confirma que `lines` é mesmo um vetor antes de tentar iterar — se viesse um texto ali, iterar daria erro.

### 4.1 Usando no Terminal

```
LABSTUDY>SET json = "{""sender"":""Dr. Souza"",""subject"":""Result review"",""lines"":[""First paragraph."",""Second paragraph.""]}"

LABSTUDY>SET id = ##class(LabStudy.Demo.Message).FromJSON(json)

LABSTUDY>WRITE id, !
1

LABSTUDY>SET m = ##class(LabStudy.Demo.Message).%OpenId(id)

LABSTUDY>DO m.PrintBody()
First paragraph.
Second paragraph.

LABSTUDY>WRITE m.Body.Size, !
36

LABSTUDY>SET m.InternalNote = "do not forward"

LABSTUDY>DO m.%JSONExport()
{"sender":"Dr. Souza","subject":"Result review"}

LABSTUDY>WRITE m.ToSummary().%ToJSON(), !
{"sender":"Dr. Souza","subject":"Result review","bodySize":36,"hasNote":1}
```

O que observar:

- O `%JSONExport()` **não** trouxe `InternalNote` (bloqueada por `%JSONINCLUDE = "none"`) nem `Body` (o adaptador não projeta streams automaticamente). Os nomes saíram em minúsculas por causa do `%JSONFIELDNAME`.
- `m.ToSummary().%ToJSON()` encadeia duas chamadas: o método devolve um `%DynamicObject`, e sobre ele chamamos `%ToJSON()`. Encadear é natural quando o retorno é um objeto.
- `m.Body.Size` é acessado **sem parênteses**, porque `Size` é propriedade, não método. Escrever `m.Body.Size()` daria erro.

### 4.2 O erro clássico do stream

```
LABSTUDY>SET s = ##class(%Stream.GlobalCharacter).%New()

LABSTUDY>DO s.Write("hello world")

LABSTUDY>WRITE s.Read(100), !


LABSTUDY>WRITE s.Size, !
11

LABSTUDY>DO s.Rewind()

LABSTUDY>WRITE s.Read(100), !
hello world
```

A primeira leitura devolveu **vazio**, apesar de o stream ter 11 caracteres. O cabeçote estava no fim. Depois do `Rewind()`, tudo aparece.

Guarde esta cena. Quando um stream parecer vazio e o `Size` disser o contrário, a resposta é sempre a mesma: faltou rebobinar.

---

## 5. Variações e detalhes

### 5.1 Estrutura dinâmica não é gravável direto

Você **não pode** declarar `Property Data As %DynamicObject;` numa classe persistente e esperar que o IRIS grave a estrutura em disco de forma útil. Estruturas dinâmicas vivem na memória.

Duas soluções comuns:

1. **Converter para texto e gravar numa string ou stream:**

```objectscript
Property RawData As %Stream.GlobalCharacter;
```

```objectscript
do ..RawData.Clear()
do ..RawData.Write(dynObject.%ToJSON())
```

E na volta:

```objectscript
do ..RawData.Rewind()
set dynObject = ##class(%DynamicObject).%FromJSON(..RawData)
```

2. **Extrair os campos para propriedades declaradas**, que é o caminho recomendado quando o formato é conhecido: você ganha validação, índices e SQL.

### 5.2 `%Get` com padrão e a diferença para `%IsDefined`

```objectscript
set obj = {"a":null}

write obj.%Get("a"), !              // vazio
write obj.%Get("zzz"), !            // vazio
write obj.%Get("zzz", "default"), ! // default
write obj.%IsDefined("a"), !        // 1
write obj.%IsDefined("zzz"), !      // 0
write obj.%GetTypeOf("a"), !        // null
write obj.%GetTypeOf("zzz"), !      // unassigned
```

Ou seja: `%Get()` sozinho **não distingue** "existe e é nulo" de "não existe". Quando essa distinção importa, use `%IsDefined()` ou `%GetTypeOf()`.

### 5.3 Números, booleanos e o formato de saída

O JSON tem tipos; o ObjectScript é mais solto. Isso gera duas surpresas comuns:

```objectscript
set obj = {}
set obj.a = 10          // sai como número:  {"a":10}
set obj.b = "10"        // sai como texto:   {"b":"10"}
set obj.c = 1           // sai como número:  {"c":1}
```

Se você quer que um campo saia como **booleano** `true`/`false`, e não como `1`/`0`, use a sintaxe literal ou o valor apropriado:

```objectscript
set obj = {"active": true}
```

E para forçar `null` explicitamente:

```objectscript
set obj = {"phone": null}
```

Como forçar um tipo específico via `%Set` em todos os casos tem particularidades por versão: **verificar na documentação oficial** quando o formato exato importar para um sistema parceiro.

### 5.4 JSON grande: use stream, não string

Se o JSON pode ser grande, evite construir uma string gigante:

```objectscript
set stream = ##class(%Stream.GlobalCharacter).%New()
do dynObject.%ToJSON(stream)
```

E na leitura:

```objectscript
set obj = ##class(%DynamicObject).%FromJSON(stream)
```

`%FromJSON` aceita string, stream ou caminho de arquivo. Trabalhar com stream evita o teto de tamanho de string e consome menos memória.

### 5.5 Limite de tamanho de string

Uma string do ObjectScript tem um limite máximo na casa dos milhões de caracteres (com o recurso de *long strings* habilitado, que é o padrão no IRIS). Ultrapassar esse limite causa o erro **`<MAXSTRING>`**.

O número exato varia com a configuração da instância: **verificar na documentação oficial**. O que importa para a prova e para a prática é o critério de decisão: **conteúdo potencialmente grande ou de tamanho desconhecido → stream, não string.**

### 5.6 Copiando e concatenando streams

```objectscript
set destino = ##class(%Stream.GlobalCharacter).%New()
set sc = destino.CopyFrom(origem)
```

`CopyFrom` copia o conteúdo inteiro. Se você quiser **acrescentar** em vez de substituir, mova para o fim antes:

```objectscript
do destino.MoveToEnd()
do origem.Rewind()
while 'origem.AtEnd {
    do destino.Write(origem.Read(8000))
}
```

### 5.7 Lendo linha a linha

```objectscript
do stream.Rewind()
while 'stream.AtEnd {
    set line = stream.ReadLine()
    write "> ", line, !
}
```

`ReadLine()` lê até o terminador de linha. O terminador usado pode ser configurado pela propriedade `LineTerminator` do stream — útil quando você lê um arquivo gerado em outro sistema operacional.

---

## 6. Pegadinhas e erros comuns

**1) Ler um stream sem rebobinar.**
Sintoma: leitura devolve vazio, mas `Size` mostra conteúdo. Solução: `Rewind()` antes de ler.

**2) Escrever num stream reaberto sem `MoveToEnd()`.**
Sintoma: o conteúdo antigo é sobrescrito a partir do início. Solução: `MoveToEnd()` antes de escrever.

**3) Usar `Size` e `AtEnd` com parênteses.**
São **propriedades**, não métodos. `stream.Size` e `stream.AtEnd`, sem parênteses.

**4) Achar que `%DynamicArray` começa em 1.**
Começa em **0**. Já `list Of` de uma classe começa em **1**. Duas convenções, duas fontes de erro.

**5) Esquecer de dobrar as aspas dentro de uma string JSON literal.**
`"{"name":"Maria"}"` não compila. O correto é `"{""name"":""Maria""}"`.

**6) Escrever `"campo": variavel` num literal JSON esperando o valor da variável.**
Sem parênteses, vai a palavra literal. O correto é `"campo": (variavel)`.

**7) Confundir `null` com inexistente.**
`%Get()` devolve vazio nos dois casos. Use `%IsDefined()` ou `%GetTypeOf()` para distinguir.

**8) Esperar que `%JSONExport()` inclua streams.**
O adaptador não projeta propriedades de stream automaticamente. Se o conteúdo precisa ir no JSON, converta você mesmo.

**9) Colocar `%JSON.Adaptor` antes de `%Persistent` na herança.**
`Extends (%JSON.Adaptor, %Persistent)` faz do adaptador a superclasse primária e quebra o armazenamento. A classe persistente vem **primeiro**.

**10) Tentar gravar um `%DynamicObject` como propriedade persistente.**
Estruturas dinâmicas vivem na memória. Serialize com `%ToJSON()` para uma string ou stream, ou extraia para propriedades declaradas.

**11) Iterar um campo sem confirmar que ele é vetor.**
Se o JSON de origem mandar um texto onde você esperava um vetor, `%GetIterator()` sobre ele falha. Confira com `%GetTypeOf()`.

**12) Construir JSON gigante como string.**
Risco de `<MAXSTRING>`. Use `%ToJSON(stream)`.

**13) Usar `%FromJSON` sem tratar JSON inválido.**
Um texto malformado gera erro em execução. No Capítulo 20 (tratamento de erros) você aprenderá a envolver isso em `TRY`/`CATCH`; por ora, saiba que o risco existe.

**14) Confundir o stream do banco com o de arquivo.**
`%Stream.GlobalCharacter` guarda dentro do banco e é gravado junto com o objeto. `%Stream.FileCharacter` precisa de `LinkToFile()` e grava num arquivo do sistema operacional.

---

## 7. MÃO NA MASSA

---

### Exercício 4.1 — Primeiros passos com estruturas dinâmicas

**a) Enunciado:** No Terminal, sem criar classe nenhuma:

1. Crie um objeto dinâmico com os campos `name`, `age`, `active` e `tags` (um vetor com três textos).
2. Imprima o JSON resultante.
3. Acrescente um campo `city` depois de criado.
4. Remova o campo `age`.
5. Imprima o tipo de cada campo, incluindo um campo inexistente.
6. Imprima o tamanho do objeto e o segundo elemento do vetor.

**b) Dica:** Vetores dinâmicos começam em zero. `%GetTypeOf` de campo inexistente devolve `unassigned`.

**c) Como testar:** Compare cada saída com o que você esperava antes de rodar.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

```
LABSTUDY>SET p = {"name":"Maria Silva","age":36,"active":true,"tags":["vip","outpatient","priority"]}

LABSTUDY>WRITE p.%ToJSON(), !
{"name":"Maria Silva","age":36,"active":true,"tags":["vip","outpatient","priority"]}

LABSTUDY>SET p.city = "Potirendaba"

LABSTUDY>DO p.%Remove("age")

LABSTUDY>WRITE p.%ToJSON(), !
{"name":"Maria Silva","active":true,"tags":["vip","outpatient","priority"],"city":"Potirendaba"}

LABSTUDY>WRITE p.%GetTypeOf("name"), !
string
LABSTUDY>WRITE p.%GetTypeOf("active"), !
boolean
LABSTUDY>WRITE p.%GetTypeOf("tags"), !
array
LABSTUDY>WRITE p.%GetTypeOf("age"), !
unassigned

LABSTUDY>WRITE p.%Size(), !
4

LABSTUDY>WRITE p.tags.%Get(1), !
outpatient

LABSTUDY>WRITE p.tags.%Size(), !
3
```

**Por que cada decisão:**

- O literal `{...}` já cria a estrutura inteira, inclusive o vetor aninhado, numa linha só. É a forma mais legível de montar JSON no ObjectScript.
- `SET p.city = "..."` acrescentou um campo que não existia. Estruturas dinâmicas não têm formulário fixo — esse é o ponto delas.
- Depois do `%Remove("age")`, o tipo de `age` virou `unassigned`. Antes era `number`.
- `p.tags.%Get(1)` devolveu o **segundo** elemento, `outpatient`. Se você esperava `vip`, releia a seção 3.3: vetores dinâmicos começam em zero.
- `%Size()` do objeto conta os campos de primeiro nível: `name`, `active`, `tags` e `city` — quatro. O vetor aninhado conta como **um** campo, independentemente de quantos itens tenha.

---

### Exercício 4.2 — Interpretando um JSON desconhecido

**a) Enunciado:** Crie `LabStudy.Demo.JsonTool` com um `ClassMethod Inspect(json)` que:

1. Converte o texto em estrutura.
2. Percorre todos os campos de primeiro nível com um iterador, imprimindo nome, tipo e valor.
3. Quando o campo for um vetor, imprime também cada elemento indentado.
4. Quando o campo for um objeto, imprime o JSON dele.

Teste com um JSON que tenha texto, número, booleano, nulo, vetor e objeto aninhado.

**b) Dica:** Dentro do laço, use `%GetTypeOf(key)` para decidir o que fazer. Para vetores e objetos aninhados, a variável `value` já vem como a estrutura correspondente.

**c) Como testar:** Cada campo deve aparecer com o tipo certo.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

`src/LabStudy/Demo/JsonTool.cls`:

```objectscript
/// Utilities to explore unknown JSON payloads.
Class LabStudy.Demo.JsonTool Extends %RegisteredObject
{

/// Prints the structure of a JSON string, field by field.
ClassMethod Inspect(json As %String) As %Status
{
    set data = ##class(%DynamicObject).%FromJSON(json)

    write "Fields: ", data.%Size(), !
    write "-------------------------------", !

    set iterator = data.%GetIterator()
    while iterator.%GetNext(.key, .value) {
        set type = data.%GetTypeOf(key)
        write key, " [", type, "]"

        if type = "array" {
            write !
            set inner = value.%GetIterator()
            while inner.%GetNext(.index, .item) {
                write "    ", index, ": ", item, !
            }
        } elseif type = "object" {
            write " -> ", value.%ToJSON(), !
        } elseif type = "null" {
            write " -> (json null)", !
        } else {
            write " -> ", value, !
        }
    }

    write "-------------------------------", !
    quit $$$OK
}

}
```

```
LABSTUDY>SET j = "{""name"":""Maria"",""age"":36,""active"":true,""phone"":null,""tags"":[""a"",""b""],""address"":{""city"":""Potirendaba""}}"

LABSTUDY>DO ##class(LabStudy.Demo.JsonTool).Inspect(j)
Fields: 6
-------------------------------
name [string] -> Maria
age [number] -> 36
active [boolean] -> 1
phone [null] -> (json null)
tags [array]
    0: a
    1: b
address [object] -> {"city":"Potirendaba"}
-------------------------------
```

**Por que cada decisão:**

- **`%GetTypeOf` antes de decidir o que fazer.** Um JSON de origem externa não é confiável: tratar cada tipo explicitamente é o que separa código robusto de código que quebra no primeiro payload diferente.
- **O tratamento separado de `null`.** Sem ele, o campo `phone` apareceria como `-> ` seguido de nada, indistinguível de uma string vazia. Com ele, a intenção fica clara na saída.
- **Repare que `active` imprimiu `1`, não `true`.** O tipo é `boolean`, mas o **valor** que o ObjectScript entrega é `1`. Essa diferença entre o tipo JSON e a representação em ObjectScript é sutil e vale ouro na prova.
- **Os índices do vetor saíram `0` e `1`.** Mais uma confirmação da indexação a partir de zero.
- **Iterador aninhado** — cada estrutura tem o seu próprio iterador. Não dá para reutilizar o de fora.

---

### Exercício 4.3 — `%JSON.Adaptor`

**a) Enunciado:** Crie `LabStudy.Demo.Supplier`, persistente e com adaptador JSON, contendo:

- `CompanyName As %String(MAXLEN = 120, %JSONFIELDNAME = "companyName")`, obrigatória;
- `TaxId As %String(MAXLEN = 20, %JSONFIELDNAME = "taxId")`;
- `ContactEmail As %String(MAXLEN = 120, %JSONFIELDNAME = "email")`;
- `CreditLimit As %Numeric(SCALE = 2, %JSONINCLUDE = "none")` — informação interna, nunca sai nem entra por JSON;
- `Active As %Boolean [ InitialExpression = 1 ]`.

No Terminal: crie um fornecedor, exporte para tela e para string, e depois **importe** um JSON diferente sobre um objeto novo, provando que `CreditLimit` é ignorado mesmo se vier no JSON.

**b) Dica:** Herança múltipla, com `%Persistent` primeiro.

**c) Como testar:** O `%JSONExport()` não deve mostrar `CreditLimit`. O `%JSONImport()` de um JSON que contenha `CreditLimit` deve deixar a propriedade zerada.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

`src/LabStudy/Demo/Supplier.cls`:

```objectscript
/// A supplier, exportable and importable as JSON.
Class LabStudy.Demo.Supplier Extends (%Persistent, %JSON.Adaptor)
{

Property CompanyName As %String(MAXLEN = 120, %JSONFIELDNAME = "companyName") [ Required ];

Property TaxId As %String(MAXLEN = 20, %JSONFIELDNAME = "taxId");

Property ContactEmail As %String(MAXLEN = 120, %JSONFIELDNAME = "email");

/// Internal only: never exported, never imported.
Property CreditLimit As %Numeric(SCALE = 2, %JSONINCLUDE = "none");

Property Active As %Boolean [ InitialExpression = 1 ];

}
```

```
LABSTUDY>SET s = ##class(LabStudy.Demo.Supplier).%New()
LABSTUDY>SET s.CompanyName = "MedParts Ltda"
LABSTUDY>SET s.TaxId = "12.345.678/0001-90"
LABSTUDY>SET s.ContactEmail = "vendas@medparts.example"
LABSTUDY>SET s.CreditLimit = 50000

LABSTUDY>DO s.%JSONExport()
{"companyName":"MedParts Ltda","taxId":"12.345.678/0001-90","email":"vendas@medparts.example","Active":true}

LABSTUDY>SET sc = s.%JSONExportToString(.text)
LABSTUDY>WRITE $$$ISOK(sc), " -> ", text, !
1 -> {"companyName":"MedParts Ltda","taxId":"12.345.678/0001-90","email":"vendas@medparts.example","Active":true}

LABSTUDY>SET sc = s.%Save()

LABSTUDY>SET incoming = "{""companyName"":""LabGlass SA"",""taxId"":""99.888.777/0001-11"",""email"":""contato@labglass.example"",""CreditLimit"":999999}"

LABSTUDY>SET novo = ##class(LabStudy.Demo.Supplier).%New()
LABSTUDY>SET sc = novo.%JSONImport(incoming)

LABSTUDY>WRITE $$$ISOK(sc), !
1

LABSTUDY>WRITE novo.CompanyName, " / ", novo.TaxId, !
LabGlass SA / 99.888.777/0001-11

LABSTUDY>WRITE "credit=[", novo.CreditLimit, "]", !
credit=[]
```

**Por que cada decisão:**

- **`%JSONFIELDNAME` em cada campo público.** A convenção do IRIS é `PascalCase`; a convenção da maioria das APIs é `camelCase`. Declarar o nome externo permite que cada mundo mantenha o seu estilo sem ninguém ceder.
- **`Active` saiu como `Active` e como `true`.** Não declaramos `%JSONFIELDNAME` para ela, então o nome da propriedade foi usado tal e qual — o que evidencia, por contraste, o efeito do parâmetro nas outras. E o valor `1` foi exportado como booleano `true` porque a propriedade é `%Boolean`: o adaptador respeita o tipo declarado.
- **`CreditLimit` com `%JSONINCLUDE = "none"` foi ignorada nas duas direções.** Ela não saiu na exportação e não entrou na importação, mesmo estando no JSON recebido. Essa é a defesa correta contra um cliente malicioso que tenta injetar campos que não deveria controlar — e é bem mais confiável do que lembrar de filtrar campo por campo no código.
- **`%JSONExportToString(.text)`** usa parâmetro de saída, com o ponto. O valor de retorno é o `%Status`, não o texto. Confundir os dois é erro comum.

---

### Exercício 4.4 — Streams

**a) Enunciado:** Crie `LabStudy.Demo.LogFile`, persistente, com `Name As %String(MAXLEN = 100)` e `Content As %Stream.GlobalCharacter`. Escreva métodos:

1. `Method Append(line)` — acrescenta uma linha ao final, com segurança.
2. `Method Dump()` — imprime todo o conteúdo lendo em blocos de 16 caracteres.
3. `Method LineCount() As %Integer` — conta as linhas usando `ReadLine()`.
4. `Method Reset()` — esvazia o conteúdo.

No Terminal: crie o objeto, acrescente cinco linhas, imprima, conte, grave, reabra pelo ID e prove que tudo continua lá.

**b) Dica:** Todo método que **lê** precisa de `Rewind()` no início. Todo método que **escreve** precisa de `MoveToEnd()`.

**c) Como testar:** Depois do `%OpenId()`, `Dump()` deve mostrar as cinco linhas e `LineCount()` deve devolver 5.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

`src/LabStudy/Demo/LogFile.cls`:

```objectscript
/// Demonstrates a character stream stored inside a persistent object.
Class LabStudy.Demo.LogFile Extends %Persistent
{

Property Name As %String(MAXLEN = 100);

Property Content As %Stream.GlobalCharacter;

/// Appends one line at the end of the stream.
Method Append(line As %String) As %Status
{
    do ..Content.MoveToEnd()
    do ..Content.WriteLine(line)
    quit $$$OK
}

/// Prints the whole content in small chunks.
Method Dump() As %Status
{
    do ..Content.Rewind()
    while '..Content.AtEnd {
        write ..Content.Read(16)
    }
    quit $$$OK
}

/// Counts how many lines the stream holds.
Method LineCount() As %Integer
{
    do ..Content.Rewind()
    set count = 0
    while '..Content.AtEnd {
        do ..Content.ReadLine()
        set count = count + 1
    }
    quit count
}

/// Empties the stream.
Method Reset() As %Status
{
    do ..Content.Clear()
    quit $$$OK
}

}
```

```
LABSTUDY>SET log = ##class(LabStudy.Demo.LogFile).%New()
LABSTUDY>SET log.Name = "import 2026-08-19"

LABSTUDY>FOR i=1:1:5 { DO log.Append("line number "_i) }

LABSTUDY>WRITE log.Content.Size, !
70

LABSTUDY>DO log.Dump()
line number 1
line number 2
line number 3
line number 4
line number 5

LABSTUDY>WRITE log.LineCount(), !
5

LABSTUDY>SET sc = log.%Save()
LABSTUDY>SET id = log.%Id()
LABSTUDY>SET log = ""

LABSTUDY>SET log = ##class(LabStudy.Demo.LogFile).%OpenId(id)
LABSTUDY>WRITE log.Name, " (", log.Content.Size, " chars)", !
import 2026-08-19 (70 chars)

LABSTUDY>DO log.Dump()
line number 1
line number 2
line number 3
line number 4
line number 5

LABSTUDY>DO log.Reset()
LABSTUDY>WRITE log.Content.Size, !
0
```

**Por que cada decisão:**

- **`MoveToEnd()` no `Append` e `Rewind()` no `Dump` e no `LineCount`.** Sem essa disciplina, os métodos passam a depender da ordem em que foram chamados. Com ela, cada método é autossuficiente: pode ser chamado a qualquer momento, em qualquer ordem, e funciona. Isso é desenho, não paranoia.
- **`LineCount` deixa o cabeçote no fim** depois de rodar. Como o `Dump` rebobina no início, isso não causa problema — precisamente porque seguimos a regra acima.
- **Nunca fizemos `%New()` do stream.** O IRIS criou automaticamente na primeira vez que `..Content` foi tocado.
- **O tamanho `70` inclui os terminadores de linha.** Cinco linhas de 13 caracteres seriam 65; os 5 restantes são os terminadores. Detalhe pequeno, mas útil para entender que `Size` conta tudo o que está no stream.
- **Depois do `%OpenId()`, o conteúdo veio inteiro.** O stream foi gravado junto com o objeto, sem nenhum código extra.

---

### Exercício 4.5 — PROJETO CONTÍNUO: laudos e integração JSON

**a) Enunciado:** Evolua o sistema do laboratório:

1. Em `LabStudy.Exam`, acrescente `Property Report As %Stream.GlobalCharacter` (o laudo em texto), com:
   - `Method AddReportLine(text)` que acrescenta uma linha ao laudo;
   - `Method PrintReport()` que imprime o laudo inteiro;
   - `Method ReportSize() As %Integer` que devolve o tamanho do laudo.
2. Faça `LabStudy.Patient` herdar também de `%JSON.Adaptor`, com nomes de campo em `camelCase`, e a propriedade `Active` marcada como `%JSONINCLUDE = "outputonly"` (o cliente não pode reativar um paciente por JSON).
3. Em `LabStudy.Patient`, acrescente `Method ToDynamicObject() As %DynamicObject` que monta um objeto dinâmico completo com os dados do paciente, o endereço, a lista de alergias e um vetor com os exames (código, valor, unidade e tamanho do laudo).
4. Em `LabStudy.Exam`, acrescente `ClassMethod ImportBatch(json) As %Integer` que recebe um JSON no formato abaixo, cria os exames para o paciente indicado e devolve quantos foram criados:

```json
{
  "recordNumber": "REG-001",
  "exams": [
    { "testCode": "HGB", "value": 13.5, "unit": "g/dL", "report": "Within reference range." },
    { "testCode": "GLU", "value": 92,   "unit": "mg/dL" }
  ]
}
```

5. Suba `LabStudy.App` para `"0.5"` e acrescente `ClassMethod ExportPatient(id)` que imprime o JSON completo do paciente.

**b) Dica:** Para achar o paciente pelo número de registro, use o `FindByRecord` do Capítulo 2. Cada exame do vetor pode não ter o campo `report` — use `%IsDefined` antes.

**c) Como testar:**

```
LABSTUDY>DO ##class(LabStudy.Patient).%KillExtent()
LABSTUDY>DO ##class(LabStudy.Exam).%KillExtent()
LABSTUDY>SET id = ##class(LabStudy.Patient).Create("Maria Silva","1990-05-17","REG-001","F")
LABSTUDY>SET n = ##class(LabStudy.Exam).ImportBatch(json)
LABSTUDY>DO ##class(LabStudy.App).ExportPatient(id)
```

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

Acrescente a `src/LabStudy/Exam.cls`:

```objectscript
/// Free text report produced for this exam.
Property Report As %Stream.GlobalCharacter;

/// Appends one line to the report.
Method AddReportLine(text As %String) As %Status
{
    do ..Report.MoveToEnd()
    do ..Report.WriteLine(text)
    quit $$$OK
}

/// Prints the whole report.
Method PrintReport() As %Status
{
    do ..Report.Rewind()
    while '..Report.AtEnd {
        write ..Report.Read(64)
    }
    quit $$$OK
}

/// Size of the report in characters.
Method ReportSize() As %Integer [ CodeMode = expression ]
{
..Report.Size
}

/// Imports a batch of exams for one patient, given as JSON.
/// Returns how many exams were created.
ClassMethod ImportBatch(json As %String) As %Integer
{
    set data = ##class(%DynamicObject).%FromJSON(json)

    if 'data.%IsDefined("recordNumber") {
        write "Missing 'recordNumber' in payload", !
        quit 0
    }

    set patient = ##class(LabStudy.Patient).FindByRecord(data.recordNumber)
    if '$ISOBJECT(patient) {
        write "Patient not found for record ", data.recordNumber, !
        quit 0
    }

    if data.%GetTypeOf("exams") '= "array" {
        write "Field 'exams' must be an array", !
        quit 0
    }

    set created = 0
    set iterator = data.exams.%GetIterator()

    while iterator.%GetNext(.index, .item) {
        if 'item.%IsDefined("testCode") {
            write "Skipping entry ", index, ": no testCode", !
            continue
        }

        set exam = ..%New(item.testCode)
        set exam.ResultValue = item.%Get("value", "")
        set exam.Unit = item.%Get("unit", "")
        set exam.Patient = patient

        if item.%IsDefined("report") {
            do exam.AddReportLine(item.report)
        }

        set sc = exam.%Save()
        if $$$ISERR(sc) {
            write "Entry ", index, " failed:", !
            do $SYSTEM.Status.DisplayError(sc)
            continue
        }

        set created = created + 1
    }

    quit created
}
```

Altere o cabeçalho de `src/LabStudy/Patient.cls` e acrescente os parâmetros JSON e o novo método:

```objectscript
Class LabStudy.Patient Extends (%Persistent, %JSON.Adaptor) [ SqlTableName = PATIENT ]
{

Property Name As %String(MAXLEN = 100, %JSONFIELDNAME = "name") [ Required ];

Property BirthDate As %Date(%JSONFIELDNAME = "birthDate") [ Required ];

Property RecordNumber As %String(MAXLEN = 20, %JSONFIELDNAME = "recordNumber") [ Required ];

Property Sex As %String(VALUELIST = ",M,F", DISPLAYLIST = ",Male,Female", %JSONFIELDNAME = "sex");

/// Clients may read this flag but never set it.
Property Active As %Boolean(%JSONINCLUDE = "outputonly") [ InitialExpression = 1 ];
```

E acrescente o método:

```objectscript
/// Builds a complete dynamic representation of this patient.
Method ToDynamicObject() As %DynamicObject
{
    set result = {}
    set result.id = ..%Id()
    set result.name = ..Name
    set result.recordNumber = ..RecordNumber
    set result.birthDate = $ZDATE(..BirthDate, 3)
    set result.age = ..Age
    set result.sex = ..Sex
    set result.active = $SELECT(..Active: "true", 1: "false")

    set result.address = {}
    set result.address.city = ..Address.City
    set result.address.state = ..Address.State

    set result.allergies = []
    for i = 1:1:..Allergies.Count() {
        do result.allergies.%Push(..Allergies.GetAt(i))
    }

    set result.exams = []
    for i = 1:1:..Exams.Count() {
        set exam = ..Exams.GetAt(i)
        set entry = {}
        set entry.testCode = exam.TestCode
        set entry.value = exam.ResultValue
        set entry.unit = exam.Unit
        set entry.reportSize = exam.ReportSize()
        do result.exams.%Push(entry)
    }

    quit result
}
```

E em `src/LabStudy/App.cls`:

```objectscript
Parameter VERSION = "0.5";

/// Prints the full JSON representation of one patient.
ClassMethod ExportPatient(id As %String) As %Status
{
    set patient = ##class(LabStudy.Patient).%OpenId(id)
    if '$ISOBJECT(patient) {
        write "Patient not found: ", id, !
        quit $$$OK
    }

    write patient.ToDynamicObject().%ToJSON(), !
    quit $$$OK
}
```

Execução esperada:

```
LABSTUDY>SET id = ##class(LabStudy.Patient).Create("Maria Silva","1990-05-17","REG-001","F")

LABSTUDY>SET p = ##class(LabStudy.Patient).%OpenId(id)
LABSTUDY>SET p.Address.City = "Potirendaba", p.Address.State = "SP"
LABSTUDY>DO p.AddAllergy("penicillin")
LABSTUDY>SET sc = p.%Save()

LABSTUDY>SET json = "{""recordNumber"":""REG-001"",""exams"":[{""testCode"":""HGB"",""value"":13.5,""unit"":""g/dL"",""report"":""Within reference range.""},{""testCode"":""GLU"",""value"":92,""unit"":""mg/dL""}]}"

LABSTUDY>WRITE ##class(LabStudy.Exam).ImportBatch(json), " exams created", !
2 exams created

LABSTUDY>DO ##class(LabStudy.App).ExportPatient(id)
{"id":"1","name":"Maria Silva","recordNumber":"REG-001","birthDate":"1990-05-17","age":36,"sex":"F","active":"true","address":{"city":"Potirendaba","state":"SP"},"allergies":["penicillin"],"exams":[{"testCode":"HGB","value":13.5,"unit":"g/dL","reportSize":24},{"testCode":"GLU","value":92,"unit":"mg/dL","reportSize":0}]}

LABSTUDY>SET e = ##class(LabStudy.Exam).%OpenId(1)
LABSTUDY>DO e.PrintReport()
Within reference range.
```

**Por que cada decisão:**

- **Duas formas de gerar JSON convivem no mesmo projeto, de propósito.** O `%JSON.Adaptor` dá uma exportação automática, plana, das propriedades declaradas — ótima para uma API de CRUD simples. O `ToDynamicObject()` monta uma representação **rica**, com idade calculada, endereço aninhado, alergias e exames. Saber quando usar cada uma é a lição: adaptador para o automático, estrutura dinâmica para o sob medida.
- **`%JSONINCLUDE = "outputonly"` em `Active`.** Um cliente externo pode ler se o paciente está ativo, mas não pode reativá-lo mandando `"Active": true`. Regra de segurança expressa na declaração, não perdida no meio do código.
- **`ImportBatch` valida em três níveis antes de tocar no banco:** o campo obrigatório existe, o paciente existe, e `exams` é mesmo um vetor. Só então entra no laço. Dados que vêm de fora nunca são confiáveis.
- **`continue` em vez de abortar o lote.** Se uma entrada do vetor estiver ruim, as outras ainda são importadas, e o problema é relatado. Um lote de cem exames não deve ser perdido porque o exame 37 veio sem código. (No Capítulo 6 você aprenderá transações, e poderá decidir conscientemente entre "tudo ou nada" e "o que der".)
- **`..%New(item.testCode)`** aproveita o `%OnNew` que escrevemos no Capítulo 3. O trabalho de lá está pagando dividendos aqui.
- **`item.%Get("value", "")` com padrão** — o campo `unit` pode faltar, e faltar não é erro. Padrão explícito evita `<UNDEFINED>`.
- **`reportSize` no JSON, em vez do laudo inteiro.** Colocar streams grandes dentro de um JSON de listagem é receita para respostas gigantes. O padrão profissional é devolver o tamanho (ou um link) e deixar o conteúdo para uma chamada específica.
- **`$SELECT(..Active: "true", 1: "false")`** produz o texto. Repare que isso gera `"active":"true"` — com aspas, ou seja, uma **string**, não um booleano JSON. É uma imperfeição consciente do exercício: serve para você **ver** a diferença entre o tipo JSON e a representação em ObjectScript, que discutimos na seção 5.3. Para um booleano JSON de verdade, o caminho é montar o literal com o valor apropriado ou usar o adaptador, que respeita o tipo `%Boolean` declarado.

---

## 8. Quiz do capítulo

**Q1.** O que a linha `set data = {"a":1,"b":[2,3]}` cria?

- A) Uma string com esse texto.
- B) Um `%DynamicObject` cujo campo `b` é um `%DynamicArray`.
- C) Um erro de compilação.
- D) Uma classe persistente.

---

**Q2.** Dado `set list = [10,20,30]`, o que `list.%Get(1)` devolve?

- A) `10`
- B) `20`
- C) `30`
- D) Vazio, porque a indexação começa em 1.

---

**Q3.** Qual método converte um texto JSON em estrutura dinâmica?

- A) `%ToJSON()`
- B) `%FromJSON()`
- C) `%JSONImport()`
- D) `%GetIterator()`

---

**Q4.** Como distinguir um campo que veio como `null` no JSON de um campo que não existe?

- A) Com `%Get()`, que devolve valores diferentes.
- B) Com `%GetTypeOf()`, que devolve `null` num caso e `unassigned` no outro.
- C) Não há como distinguir.
- D) Com `%Size()`.

---

**Q5.** Qual é a assinatura correta para percorrer um `%DynamicObject`?

- A) `while obj.%GetNext(.key, .value) { }`
- B) `set it = obj.%GetIterator() while it.%GetNext(.key, .value) { }`
- C) `for i = 1:1:obj.%Size() { set v = obj.%Get(i) }`
- D) `set it = obj.%Iterator() while it.%Next() { }`

---

**Q6.** Uma classe deve exportar e importar JSON automaticamente e ser persistente. Qual declaração está correta?

- A) `Class X Extends (%JSON.Adaptor, %Persistent)`
- B) `Class X Extends (%Persistent, %JSON.Adaptor)`
- C) `Class X Extends %JSON.Adaptor`
- D) `Class X Extends %Persistent [ JSON ]`

---

**Q7.** O que faz o parâmetro de propriedade `%JSONINCLUDE = "outputonly"`?

- A) A propriedade é exportada mas ignorada na importação.
- B) A propriedade é importada mas não exportada.
- C) A propriedade é ignorada nos dois sentidos.
- D) A propriedade vira somente leitura no objeto.

---

**Q8.** Você escreveu num stream e, em seguida, chamou `Read(100)`, que devolveu vazio, embora `Size` mostre 500. O que aconteceu?

- A) O stream não foi gravado.
- B) O cabeçote está no fim depois da escrita; falta chamar `Rewind()`.
- C) `Read` só funciona em streams binários.
- D) `Size` está errado.

---

**Q9.** `Size` e `AtEnd` de um stream são:

- A) Métodos, chamados com parênteses.
- B) Propriedades, acessadas sem parênteses.
- C) Funções do sistema, com `$`.
- D) Palavras-chave de classe.

---

**Q10.** Qual classe usar para guardar, **dentro do banco de dados**, o texto longo de um laudo?

- A) `%Stream.FileCharacter`
- B) `%Stream.GlobalBinary`
- C) `%Stream.GlobalCharacter`
- D) `%String(MAXLEN = 100000)`

---

**Q11.** Uma propriedade de stream numa classe persistente precisa de `%New()` explícito antes do primeiro uso?

- A) Sim, sempre.
- B) Não; o IRIS cria o stream automaticamente no primeiro acesso.
- C) Só em streams de arquivo.
- D) Só se a classe tiver mais de um stream.

---

**Q12.** Você reabriu um objeto com `%OpenId()` e quer **acrescentar** texto ao final de um stream já preenchido. O que fazer antes de escrever?

- A) `Rewind()`
- B) `Clear()`
- C) `MoveToEnd()`
- D) Nada; `Write` sempre acrescenta ao final.

---

**Q13.** Qual das opções é a forma correta de escrever, no código ObjectScript, a string JSON `{"name":"Maria"}`?

- A) `set j = "{"name":"Maria"}"`
- B) `set j = "{""name"":""Maria""}"`
- C) `set j = '{"name":"Maria"}'`
- D) `set j = {{"name":"Maria"}}`

---

**Q14.** Num literal dinâmico, como inserir o **valor** de uma variável `city` no campo `"city"`?

- A) `{"city": city}`
- B) `{"city": "city"}`
- C) `{"city": (city)}`
- D) `{"city": $city}`

---

### Gabarito comentado

**Q1 — Resposta: B.**
- **B está certa:** o literal com chaves cria um `%DynamicObject`; o literal com colchetes dentro dele cria um `%DynamicArray`.
- **A está errada:** não é texto; é uma estrutura viva, com métodos.
- **C está errada:** a sintaxe literal é válida no ObjectScript.
- **D está errada:** estruturas dinâmicas não são classes persistentes.

**Q2 — Resposta: B.**
- **B está certa:** `%DynamicArray` é indexado a partir de **zero**, então a posição 1 é o segundo elemento, `20`.
- **A está errada:** `10` está em `%Get(0)`.
- **C está errada:** `30` está em `%Get(2)`.
- **D está errada:** quem começa em 1 é a coleção `list Of` de uma classe, não o vetor dinâmico.

**Q3 — Resposta: B.**
- **B está certa:** `%FromJSON()` interpreta texto (ou stream, ou arquivo) e devolve a estrutura.
- **A está errada:** `%ToJSON()` faz o caminho inverso.
- **C está errada:** `%JSONImport()` pertence ao `%JSON.Adaptor` e preenche um objeto declarado, não uma estrutura dinâmica.
- **D está errada:** o iterador percorre uma estrutura já existente.

**Q4 — Resposta: B.**
- **B está certa:** `%GetTypeOf()` devolve `null` quando o campo existe com valor nulo e `unassigned` quando não existe.
- **A está errada:** `%Get()` devolve vazio nos dois casos.
- **C está errada:** a distinção existe e é importante em integrações.
- **D está errada:** `%Size()` só conta elementos.

**Q5 — Resposta: B.**
- **B está certa:** obtém-se o iterador com `%GetIterator()` e avança-se com `%GetNext(.key, .value)`, com os pontos por serem parâmetros de saída.
- **A está errada:** `%GetNext` pertence ao iterador, não ao objeto.
- **C está errada:** os campos de um objeto têm nomes, não índices sequenciais.
- **D está errada:** esses nomes de método não existem.

**Q6 — Resposta: B.**
- **B está certa:** herança múltipla com parênteses, e a classe persistente **primeiro**, por ser a superclasse primária que define o armazenamento.
- **A está errada:** invertendo a ordem, o adaptador vira superclasse primária e o armazenamento quebra.
- **C está errada:** sem `%Persistent`, a classe não grava em disco.
- **D está errada:** não existe essa palavra-chave.

**Q7 — Resposta: A.**
- **A está certa:** `outputonly` significa que a propriedade sai na exportação e é ignorada na importação.
- **B está errada:** esse é o `inputonly`.
- **C está errada:** esse é o `none`.
- **D está errada:** o parâmetro afeta apenas a projeção JSON, não o objeto.

**Q8 — Resposta: B.**
- **B está certa:** depois de escrever, a posição atual fica no fim. Ler dali não devolve nada. `Rewind()` resolve.
- **A está errada:** o `Size` prova que o conteúdo está lá.
- **C está errada:** `Read` funciona nos dois tipos de stream.
- **D está errada:** `Size` reflete corretamente o conteúdo.

**Q9 — Resposta: B.**
- **B está certa:** são propriedades: `stream.Size` e `stream.AtEnd`, sem parênteses.
- **A está errada:** usar parênteses causa erro de método inexistente.
- **C está errada:** não são funções do sistema.
- **D está errada:** não são palavras-chave.

**Q10 — Resposta: C.**
- **C está certa:** texto dentro do banco pede `%Stream.GlobalCharacter`.
- **A está errada:** `FileCharacter` guarda num arquivo do sistema operacional.
- **B está errada:** `GlobalBinary` é para bytes, não texto.
- **D está errada:** uma string tem teto de tamanho e não é a ferramenta certa para conteúdo indeterminado.

**Q11 — Resposta: B.**
- **B está certa:** o IRIS instancia o stream automaticamente no primeiro acesso à propriedade.
- **A está errada:** o `%New()` explícito funciona, mas não é necessário.
- **C está errada:** o comportamento é o mesmo.
- **D está errada:** não há relação com a quantidade de streams.

**Q12 — Resposta: C.**
- **C está certa:** `MoveToEnd()` leva a posição para o fim, e a escrita seguinte acrescenta.
- **A está errada:** `Rewind()` levaria ao início, e a escrita sobrescreveria o começo.
- **B está errada:** `Clear()` apagaria tudo.
- **D está errada:** `Write` escreve na **posição atual**, que após um `%OpenId()` é o início.

**Q13 — Resposta: B.**
- **B está certa:** dentro de uma string do ObjectScript, cada aspa dupla literal é escrita duas vezes.
- **A está errada:** a string termina na segunda aspa e o resto vira erro de sintaxe.
- **C está errada:** o ObjectScript não delimita strings com aspas simples.
- **D está errada:** essa sintaxe não existe.

**Q14 — Resposta: C.**
- **C está certa:** os parênteses indicam que o conteúdo é uma expressão a ser avaliada.
- **A está errada:** sem os parênteses, o compilador não aceita a variável ali.
- **B está errada:** gravaria a palavra literal `city`.
- **D está errada:** o cifrão é para funções e variáveis especiais do sistema.

---

## 9. Resumo relâmpago

1. JSON tem duas construções: **objeto** `{ "chave": valor }` e **vetor** `[ v1, v2 ]`. Valores: texto, número, `true`, `false`, `null`, objeto, vetor.
2. `{ }` no código cria **`%DynamicObject`**; `[ ]` cria **`%DynamicArray`**.
3. Para inserir uma expressão num literal dinâmico, envolva-a em **parênteses**: `{"a": (variavel)}`.
4. Dentro de uma string do ObjectScript, aspa dupla literal se escreve **duas vezes**.
5. Métodos de objeto dinâmico: `%Set`, `%Get(chave, padrao)`, `%Remove`, `%IsDefined`, `%Size`, `%GetTypeOf`, `%ToJSON`.
6. Métodos de vetor dinâmico: `%Get`, `%Set`, `%Push`, `%Pop`, `%Size`. **Índice começa em ZERO.**
7. `list Of` de classe começa em **1** e usa `GetAt()`. Não confunda com o vetor dinâmico.
8. `##class(%DynamicObject).%FromJSON(origem)` aceita string, stream ou caminho de arquivo.
9. `%GetTypeOf` devolve `string`, `number`, `boolean`, `null`, `object`, `array`, `oref` ou `unassigned`. É a única forma segura de distinguir **`null`** de **inexistente**.
10. Percorrer: `set it = obj.%GetIterator()` e `while it.%GetNext(.key, .value) { }`.
11. `%JSON.Adaptor` dá `%JSONExport()`, `%JSONExportToString(.str)` e `%JSONImport(json)`. Na herança, **`%Persistent` vem primeiro**.
12. Parâmetros do adaptador: `%JSONFIELDNAME` (nome externo), `%JSONINCLUDE` (`all`/`inputonly`/`outputonly`/`none`), `%JSONNULL`, `%JSONIGNORENULL`.
13. **Stream** é para conteúdo grande ou de tamanho desconhecido. String tem teto e estoura com `<MAXSTRING>`.
14. Quatro classes principais: `%Stream.GlobalCharacter`, `%Stream.GlobalBinary`, `%Stream.FileCharacter`, `%Stream.FileBinary`.
15. Ciclo do stream: **escreveu → `Rewind()` → leu.** Para acrescentar depois de reabrir: **`MoveToEnd()` antes de escrever.**
16. `Size` e `AtEnd` são **propriedades**, sem parênteses. `Read(n)`, `ReadLine()`, `Write()`, `WriteLine()`, `Clear()`, `CopyFrom()` são métodos.
17. Stream como propriedade persistente é criado automaticamente e gravado junto com o objeto.
18. Estrutura dinâmica **não é gravável direto** em disco: serialize com `%ToJSON()` ou extraia para propriedades declaradas.

---

## 10. Cartões de memorização

**Frente:** O que `{ }` e `[ ]` criam no código ObjectScript?
**Verso:** `%DynamicObject` e `%DynamicArray`, respectivamente.

**Frente:** Em que índice começa um `%DynamicArray`?
**Verso:** Em **zero**. `%Get(0)` é o primeiro elemento.

**Frente:** Em que índice começa uma coleção `list Of`?
**Verso:** Em **um**. `GetAt(1)` é o primeiro elemento.

**Frente:** Como inserir o valor de uma variável num literal dinâmico?
**Verso:** Entre parênteses: `{"city": (city)}`.

**Frente:** Como escrever uma aspa dupla dentro de uma string do ObjectScript?
**Verso:** Duplicando-a: `"{""name"":""Maria""}"`.

**Frente:** Qual método converte texto JSON em estrutura?
**Verso:** `##class(%DynamicObject).%FromJSON(origem)` — aceita string, stream ou arquivo.

**Frente:** Qual método converte estrutura em texto JSON?
**Verso:** `obj.%ToJSON()`. Passando um destino, escreve nele em vez de devolver.

**Frente:** Como distinguir `null` de campo inexistente?
**Verso:** `%GetTypeOf()` devolve `null` no primeiro caso e `unassigned` no segundo. `%Get()` devolve vazio nos dois.

**Frente:** Como percorrer todos os campos de um objeto dinâmico?
**Verso:** `set it = obj.%GetIterator()` e depois `while it.%GetNext(.key, .value) { ... }`.

**Frente:** O que preciso herdar para exportar um objeto persistente como JSON?
**Verso:** `Extends (%Persistent, %JSON.Adaptor)` — persistente **primeiro**.

**Frente:** O que faz `%JSONFIELDNAME`?
**Verso:** Define o nome que a propriedade terá no JSON, sem mudar o nome no ObjectScript.

**Frente:** O que fazem os quatro valores de `%JSONINCLUDE`?
**Verso:** `all` (padrão), `outputonly` (só exporta), `inputonly` (só importa), `none` (ignorada nos dois sentidos).

**Frente:** Por que existem streams?
**Verso:** Porque uma string tem tamanho máximo. Streams tratam conteúdo grande ou indeterminado, lido e escrito aos poucos.

**Frente:** Li um stream e veio vazio, mas `Size` mostra conteúdo. O que houve?
**Verso:** O cabeçote está no fim. Falta `Rewind()` antes de ler.

**Frente:** Reabri um objeto e quero acrescentar texto no stream. O que faço antes?
**Verso:** `MoveToEnd()`. Sem isso, a escrita começa no início e sobrescreve.

**Frente:** `Size` e `AtEnd` são métodos ou propriedades?
**Verso:** Propriedades. Acessadas sem parênteses: `stream.Size`, `stream.AtEnd`.

**Frente:** Qual classe para texto longo guardado dentro do banco?
**Verso:** `%Stream.GlobalCharacter`. Para bytes, `%Stream.GlobalBinary`.

**Frente:** Preciso de `%New()` para uma propriedade de stream?
**Verso:** Não. O IRIS a cria automaticamente no primeiro acesso, e a grava junto com o objeto.

**Frente:** Posso declarar `Property X As %DynamicObject` numa classe persistente?
**Verso:** Não de forma útil. Estruturas dinâmicas vivem na memória; serialize com `%ToJSON()` ou extraia para propriedades declaradas.

**Frente:** Qual erro aparece quando uma string passa do tamanho máximo?
**Verso:** `<MAXSTRING>`. É o sinal de que aquele conteúdo deveria estar num stream.

---

Digite CONTINUAR para o próximo capítulo.
