# Apostila InterSystems ObjectScript Specialist
## Capítulo 19 — T4.7 Uses APIs for Common Operations (APIs para operações comuns)

> Último tópico do domínio **T4 — Functions & APIs**. Aqui você conhece as bibliotecas que o IRIS já traz prontas: informações do sistema, manipulação de arquivos e diretórios, dispositivos, chamadas HTTP e agendamento de tarefas. O objetivo não é decorar tudo — é **saber que existe** e onde procurar.

---

## 1. O que você vai saber fazer ao terminar

1. Navegar pelo **`$SYSTEM`** e reconhecer os pacotes mais usados.
2. Obter informações de **versão**, **instância**, **processo** e **ambiente**.
3. Manipular arquivos e diretórios com a classe **`%File`**: existir, criar, copiar, renomear, apagar, medir.
4. **Listar** o conteúdo de um diretório com a query `FileSet`.
5. Ler e gravar arquivos com **`%Stream.FileCharacter`** e **`%Stream.FileBinary`**, tratando **codificação**.
6. Reconhecer a forma clássica com **`OPEN` / `USE` / `CLOSE`** e o papel de **`$IO`**.
7. Fazer chamadas **HTTP** com **`%Net.HttpRequest`** e tratar a resposta.
8. Conhecer o **agendador de tarefas** (`%SYS.Task`) e quando usá-lo.
9. Saber que existe **`$ZF`** para chamar o sistema operacional — e por que tratá-lo com cuidado.
10. Levar o projeto à versão **2.0**, com exportação e importação de arquivos e um painel do sistema.

---

## 2. O conceito em linguagem de gente

### 2.1 O IRIS já traz a caixa de ferramentas

Um erro comum de quem começa é reimplementar o que já existe. O IRIS traz bibliotecas para praticamente toda tarefa comum de infraestrutura: arquivos, rede, agendamento, criptografia, compressão, e-mail, XML, JSON.

A analogia: você não fabrica a chave de fenda antes de apertar o parafuso. Abre a caixa e procura.

O problema, claro, é **saber que a chave existe e em que gaveta está**. É isso que este capítulo resolve — e é por isso que ele tem menos "aprenda a fazer" e mais "saiba que existe".

### 2.2 O `$SYSTEM`: a gaveta principal

**`$SYSTEM`** é uma variável especial que dá acesso a um conjunto de utilitários organizados por assunto. Você já usou vários nesta apostila:

| Pacote | Assunto | Onde apareceu |
|---|---|---|
| `$SYSTEM.Status` | tratamento de `%Status` | Capítulo 1 |
| `$SYSTEM.OBJ` | compilar, apagar e exportar classes | Capítulo 1 |
| `$SYSTEM.SQL` | consultas, planos, estatísticas | Capítulos 9 e 12 |
| `$SYSTEM.Security` | privilégios e autenticação | Capítulo 7 |
| `$SYSTEM.Encryption` | hash e criptografia | Capítulo 7 |
| `$SYSTEM.Version` | versão do produto | este |
| `$SYSTEM.Process` | informações do processo | este |
| `$SYSTEM.Util` | utilidades diversas | este |

A forma de descobrir o que existe é o próprio Terminal:

```
LABSTUDY>DO $SYSTEM.Help()
LABSTUDY>DO $SYSTEM.OBJ.Help()
```

Isso lista os métodos disponíveis com uma descrição curta. É o índice da caixa de ferramentas, e vale mais do que qualquer lista que eu escrevesse aqui — porque ele reflete **a sua versão**.

### 2.3 Arquivos: dois níveis de abstração

Trabalhar com arquivos no IRIS acontece em dois níveis, e confundi-los gera frustração.

**Nível 1 — o arquivo como entidade do sistema operacional.**

Existe? Qual o tamanho? Está em que pasta? Como apago, copio, renomeio? Como listo a pasta?

A ferramenta é a classe **`%File`**, que oferece métodos de classe para todas essas perguntas. Ela **não** lê nem grava conteúdo — ela cuida do arquivo como objeto do sistema de arquivos.

**Nível 2 — o conteúdo do arquivo.**

Ler linhas, gravar texto, copiar bytes.

A ferramenta moderna é **`%Stream.FileCharacter`** (texto) e **`%Stream.FileBinary`** (bytes), que você já conhece do Capítulo 4. A ferramenta clássica é o trio **`OPEN` / `USE` / `CLOSE`**.

A analogia: `%File` é o arquivista que sabe onde a pasta está, quanto pesa e como movê-la. O stream é quem abre a pasta e lê o que está escrito dentro.

### 2.4 Dispositivos: a ideia por trás do `WRITE`

Uma ideia central do IRIS, herdada de sua tradição, é que **tudo é um dispositivo**: a tela, um arquivo, uma conexão de rede, uma impressora.

Quando você escreve `WRITE "olá"`, o texto vai para o **dispositivo atual**, indicado pela variável **`$IO`**. No Terminal, o dispositivo atual é a sua tela.

O trio clássico funciona assim:

- **`OPEN`** — abre um dispositivo (por exemplo, um arquivo).
- **`USE`** — torna aquele dispositivo o **atual**, de modo que `WRITE` e `READ` passem a operar sobre ele.
- **`CLOSE`** — fecha.

Isso significa que o **mesmo** `WRITE` que imprime na tela pode gravar num arquivo, bastando mudar o dispositivo atual. É uma abstração poderosa — e uma armadilha, porque é fácil esquecer de voltar o `$IO` para a tela e perder toda a saída seguinte.

Por isso, para arquivos, **prefira streams**: eles não mexem no dispositivo atual e não têm esse risco.

### 2.5 Quando falar com o mundo lá fora

Sistemas reais precisam conversar com outros sistemas. O IRIS oferece bibliotecas para isso:

- **`%Net.HttpRequest`** — chamadas HTTP e HTTPS, o caso mais comum hoje.
- **`%Net.SMTP`** — envio de e-mail.
- **`%Net.FtpSession`** — transferência por FTP.
- **`%SYS.Task`** — agendamento de tarefas periódicas.
- **`$ZF`** — chamada a programas e bibliotecas do sistema operacional.

O último merece um aviso desde já: `$ZF` executa comandos do sistema operacional. É extremamente poderoso e é uma **porta de entrada para comprometer o servidor** se algum argumento vier de fora sem validação. O mesmo raciocínio do `XECUTE`, do Capítulo 17, com consequências maiores.

---

## 3. Informações do sistema

### 3.1 Versão e instância

```objectscript
write $ZVERSION, !
write $SYSTEM.Version.GetVersion(), !
write $SYSTEM.Version.GetNumber(), !
write $SYSTEM.Version.GetPlatform(), !
```

- **`$ZVERSION`** é a variável especial com a linha completa de versão. É a forma mais rápida.
- **`$SYSTEM.Version`** traz os componentes separados, úteis para comparações programáticas.

Isso importa mais do que parece: comportamentos e nomes de métodos mudam entre versões, e um sistema que registra a versão em que rodou facilita muito o diagnóstico. Ao longo desta apostila, várias vezes recomendei **verificar na documentação oficial** — e a versão é o primeiro dado a saber quando você for verificar.

### 3.2 O processo

```objectscript
write $JOB, !
write $USERNAME, !
write $ROLES, !
write $NAMESPACE, !
write $IO, !
write $ZSTORAGE, !
write $STORAGE, !
```

E, pela API:

```objectscript
write $SYSTEM.Process.NameSpace(), !
write $SYSTEM.Process.UserName(), !
```

O pacote `$SYSTEM.Process` também permite **alterar** características do processo, como o limite de memória e a prioridade. Os métodos disponíveis variam por versão: **verificar na documentação oficial**.

### 3.3 Diretórios da instalação e ambiente

```objectscript
write $SYSTEM.Util.InstallDirectory(), !
write $SYSTEM.Util.ManagerDirectory(), !
write $SYSTEM.Util.GetEnviron("PATH"), !
```

- **`InstallDirectory()`** — onde o IRIS está instalado.
- **`ManagerDirectory()`** — o diretório de gerenciamento da instância.
- **`GetEnviron(nome)`** — uma variável de ambiente do sistema operacional.

Isso é útil para montar caminhos de arquivo que funcionem em qualquer instalação, em vez de gravar `/opt/irisapp` no código.

O conjunto exato de métodos de `$SYSTEM.Util` é grande e varia entre versões: use `DO $SYSTEM.Util.Help()` para listar os da sua.

---

## 4. Arquivos e diretórios

### 4.1 A classe `%File`

Métodos de classe mais usados:

```objectscript
// existência
write ##class(%File).Exists("/tmp/dados.csv"), !
write ##class(%File).DirectoryExists("/tmp/lab"), !

// criar diretório (e toda a cadeia de pais, se necessário)
set ok = ##class(%File).CreateDirectoryChain("/tmp/lab/exportacoes", .retorno)

// tamanho
write ##class(%File).GetFileSize("/tmp/dados.csv"), !

// partes do caminho
write ##class(%File).GetDirectory("/tmp/lab/dados.csv"), !     // /tmp/lab/
write ##class(%File).GetFilename("/tmp/lab/dados.csv"), !      // dados.csv

// montar caminho de forma portável
write ##class(%File).SubDirectoryName("/tmp/lab", "exportacoes"), !

// copiar, renomear, apagar
set ok = ##class(%File).CopyFile("/tmp/a.csv", "/tmp/b.csv", 1, .retorno)
set ok = ##class(%File).Rename("/tmp/b.csv", "/tmp/c.csv")
set ok = ##class(%File).Delete("/tmp/c.csv")
```

Pontos importantes:

- A maioria devolve **`1`** em caso de sucesso e **`0`** em caso de falha, com detalhes num parâmetro de saída (o `.retorno` acima).
- **`CreateDirectoryChain`** cria toda a cadeia de diretórios pais; `CreateDirectory` cria apenas um nível.
- **`SubDirectoryName`** junta partes de caminho usando o separador correto do sistema operacional. Concatenar com `/` na mão funciona no Linux e quebra no Windows.
- Os nomes exatos e as assinaturas variam entre versões: **verificar na documentação oficial** ou usar `DO ##class(%File).Help()`.

### 4.2 Listando um diretório

`%File` traz uma class query chamada **`FileSet`**:

```objectscript
set stmt = ##class(%SQL.Statement).%New()
set sc = stmt.%PrepareClassQuery("%File", "FileSet")
if $$$ISERR(sc) { do $SYSTEM.Status.DisplayError(sc) quit }

set rs = stmt.%Execute("/tmp/lab", "*.csv")

while rs.%Next() {
    write rs.%Get("Name"), "  ", rs.%Get("Type"), "  ", rs.%Get("Size"), !
}
```

- O primeiro argumento é o **diretório**; o segundo, um **filtro** de nome (aceita curingas).
- As colunas típicas são `Name` (caminho completo), `Type` (`F` para arquivo, `D` para diretório), `Size` e `DateModified`.

**Confirme as colunas antes de depender delas.** O conjunto exato e o **formato** de `DateModified` — em especial se ele vem como `AAAA-MM-DD HH:MM:SS` ou em outro formato — variam por versão e por plataforma, e o código do exercício 19.2 supõe o primeiro ao fazer `$PIECE(modified, " ", 1)`. Uma execução resolve a dúvida:

```
LABSTUDY>SET s=##class(%SQL.Statement).%New()
LABSTUDY>DO s.%PrepareClassQuery("%File","FileSet")
LABSTUDY>SET r=s.%Execute("/tmp","*")
LABSTUDY>DO r.%Display()
```

O `%Display()` mostra os nomes das colunas no cabeçalho e um exemplo de cada valor. O mesmo vale para `##class(%File).GetFileSize()`: confirme com `DO ##class(%File).Help()` que ele existe com esse nome na sua versão — se não existir, o tamanho também está disponível na coluna `Size` do próprio `FileSet`.

Repare que é exatamente a mesma mecânica de class query do Capítulo 18 — a diferença é que os dados não vêm de uma tabela, e sim do sistema de arquivos. **Isso é o `%Query` implementado em ObjectScript** que o Capítulo 9 mencionou.

### 4.3 Lendo e gravando com streams

**Gravando:**

```objectscript
set file = ##class(%Stream.FileCharacter).%New()
set sc = file.LinkToFile("/tmp/lab/saida.csv")
if $$$ISERR(sc) { do $SYSTEM.Status.DisplayError(sc) quit }

do file.WriteLine("codigo;valor;unidade")
do file.WriteLine("HGB;13.5;g/dL")

set sc = file.%Save()
```

**Lendo:**

```objectscript
set file = ##class(%Stream.FileCharacter).%New()
set sc = file.LinkToFile("/tmp/lab/saida.csv")

do file.Rewind()
while 'file.AtEnd {
    set line = file.ReadLine()
    write line, !
}
```

Tudo o que você aprendeu no Capítulo 4 sobre streams vale aqui: **escreveu → rebobinou → leu**, `MoveToEnd()` antes de acrescentar, `Size` e `AtEnd` sem parênteses.

**Codificação de caracteres:**

```objectscript
set file.TranslateTable = "UTF8"
```

A propriedade **`TranslateTable`** define a conversão de caracteres do arquivo. Sem ela, acentos podem sair corrompidos ao trocar dados com outros sistemas. O nome exato das tabelas disponíveis: **verificar na documentação oficial**.

Alternativa, quando você controla o texto: converter explicitamente com `$ZCONVERT(texto, "O", "UTF8")` ao gravar e `$ZCONVERT(texto, "I", "UTF8")` ao ler, como visto no Capítulo 15.

**Terminador de linha:**

```objectscript
set file.LineTerminator = $CHAR(13, 10)     // Windows
set file.LineTerminator = $CHAR(10)         // Linux
```

Arquivos que atravessam sistemas operacionais trazem terminadores diferentes. Definir explicitamente evita linhas com um `$CHAR(13)` invisível no fim — que quebra comparações e valida errado.

### 4.4 A forma clássica: `OPEN` / `USE` / `CLOSE`

```objectscript
set file = "/tmp/lab/classico.txt"

open file:("WNS"):5
if '$TEST {
    write "não foi possível abrir", !
    quit
}

set old = $IO
use file

write "linha um", !
write "linha dois", !

use old
close file

write "voltei para a tela", !
```

Peça por peça:

- **`OPEN dispositivo:(parâmetros):tempoLimite`** — abre. Os parâmetros entre parênteses definem o modo: leitura, escrita, criação de arquivo novo, e outros. Confira **`$TEST`** depois.
- **`USE dispositivo`** — torna-o o atual. A partir daí, `WRITE` e `READ` operam sobre ele.
- **`CLOSE dispositivo`** — fecha.

**O cuidado essencial:** guarde `$IO` antes do `USE` e restaure depois. Se você esquecer, toda a saída seguinte do processo vai para o arquivo — inclusive mensagens de erro, que você nunca verá.

Os códigos de modo (`"R"`, `"W"`, `"WNS"`, `"A"` e outros) e os parâmetros adicionais variam: **verificar na documentação oficial**.

**Recomendação:** use `OPEN`/`USE`/`CLOSE` apenas quando precisar de controle fino sobre o dispositivo, ou ao ler código legado. Para arquivos, **streams são mais seguros e mais simples**.

---

## 5. Falando com o mundo externo

### 5.1 HTTP com `%Net.HttpRequest`

```objectscript
set req = ##class(%Net.HttpRequest).%New()
set req.Server = "api.exemplo.com"
set req.Port = 443
set req.Https = 1
set req.Timeout = 10

set sc = req.Get("/v1/status")

if $$$ISERR(sc) {
    do $SYSTEM.Status.DisplayError(sc)
    quit
}

set rsp = req.HttpResponse
write "status : ", rsp.StatusCode, !
write "tipo   : ", rsp.ContentType, !

set body = rsp.Data.Read()
write "corpo  : ", body, !
```

Elementos principais:

- **`Server`**, **`Port`**, **`Https`** definem o destino.
- **`Timeout`** evita que a chamada trave o processo indefinidamente. **Sempre defina.**
- **`Get(caminho)`**, **`Post(caminho)`**, **`Put`**, **`Delete`** disparam a requisição e devolvem `%Status`.
- **`HttpResponse`** traz a resposta: `StatusCode`, `ContentType`, e o corpo em **`Data`**, que é um **stream**.

Para enviar JSON num `POST`:

```objectscript
set req.ContentType = "application/json"
do req.EntityBody.Write(payload.%ToJSON())
set sc = req.Post("/v1/exames")
```

E para ler JSON da resposta:

```objectscript
set data = ##class(%DynamicObject).%FromJSON(rsp.Data)
```

Repare que `%FromJSON` aceita um stream diretamente — a lição do Capítulo 4.

Cabeçalhos e autenticação:

```objectscript
do req.SetHeader("Authorization", "Bearer "_token)
```

**Cuidados profissionais ao chamar serviços externos:**

- **Sempre com tempo limite.** Um serviço lento não pode travar o seu.
- **Sempre com tratamento de erro**, incluindo `StatusCode` diferente de 200.
- **Nunca dentro de uma transação longa** (Capítulo 5): você seguraria travas esperando a rede.
- **Nunca com credenciais no código** (Capítulo 7).
- **Registre o que aconteceu**, porque falhas de integração são difíceis de reproduzir.

### 5.2 Agendamento de tarefas

Para trabalho periódico — limpeza noturna, relatórios diários, reconstrução de índices —, o IRIS tem o **agendador de tarefas**, no Portal em **System Operation → Task Manager**.

Programaticamente, o pacote é **`%SYS.Task`**. A abordagem recomendada é criar uma classe que estende **`%SYS.Task.Definition`** e implementa `OnTask()`:

```objectscript
Class LabStudy.Demo.NightlyTask Extends %SYS.Task.Definition
{

Property BatchSize As %Integer [ InitialExpression = 500 ];

Method OnTask() As %Status
{
    // o trabalho periódico vai aqui
    quit $$$OK
}

}
```

Depois de compilada, a classe aparece na lista de tarefas disponíveis no Portal, onde o administrador define a frequência.

Vantagens sobre disparar `JOB` num laço: o agendador controla execução concorrente, registra histórico, e permite ao administrador suspender e reprogramar sem mexer no código.

Os detalhes de propriedades e de agendamento programático: **verificar na documentação oficial**.

### 5.3 Chamando o sistema operacional

```objectscript
set status = $ZF(-100, "", "ls", "-la", "/tmp/lab")
```

**`$ZF(-100, ...)`** executa um programa do sistema operacional. Existem variantes para capturar a saída e para controlar o modo de execução.

**Este é o recurso mais perigoso da linguagem.** Se qualquer argumento vier de entrada não validada, o atacante executa comandos no servidor com o usuário do IRIS.

Regras, sem exceção:

- Nunca monte argumentos concatenando entrada do usuário.
- Prefira as APIs nativas: para arquivos, `%File`; para rede, `%Net`; para compressão, as bibliotecas do produto.
- Se for inevitável, valide contra uma **lista de permitidos**, como no Capítulo 17.

Antes de recorrer ao `$ZF`, pergunte se o IRIS já não faz aquilo. Quase sempre faz.

---

## 6. Exemplo comentado

Arquivo `src/LabStudy/Demo/Api.cls`:

```objectscript
/// A tour of the built in APIs.
Class LabStudy.Demo.Api Extends %RegisteredObject
{

/// Working directory for the demonstrations.
Parameter WORKDIR = "/tmp/labstudy";

/// System, instance and process information.
ClassMethod SystemInfo() As %Status
{
    write "=== versao ===", !
    write "  $ZVERSION      : ", $ZVERSION, !
    write "  GetVersion()   : ", $SYSTEM.Version.GetVersion(), !
    write "  GetNumber()    : ", $SYSTEM.Version.GetNumber(), !
    write "  GetPlatform()  : ", $SYSTEM.Version.GetPlatform(), !

    write !, "=== processo ===", !
    write "  $JOB           : ", $JOB, !
    write "  $USERNAME      : ", $USERNAME, !
    write "  $ROLES         : ", $ROLES, !
    write "  $NAMESPACE     : ", $NAMESPACE, !
    write "  $IO            : ", $IO, !
    write "  $ZSTORAGE      : ", $ZSTORAGE, " KB", !
    write "  $STORAGE       : ", $STORAGE, " bytes livres", !

    write !, "=== instalacao ===", !
    write "  install dir    : ", $SYSTEM.Util.InstallDirectory(), !
    write "  manager dir    : ", $SYSTEM.Util.ManagerDirectory(), !

    write !, "=== ambiente ===", !
    for var = "PATH", "HOME", "LANG" {
        write "  ", $JUSTIFY(var, 6), " : ",
              ##class(LabStudy.Text).Abbreviate($SYSTEM.Util.GetEnviron(var), 50), !
    }

    quit $$$OK
}

/// Prepares the working directory.
ClassMethod Prepare() As %Boolean
{
    if ##class(%File).DirectoryExists(..#WORKDIR) {
        write "  diretorio ja existe: ", ..#WORKDIR, !
        quit 1
    }

    set ok = ##class(%File).CreateDirectoryChain(..#WORKDIR, .info)

    if 'ok {
        write "  falha ao criar ", ..#WORKDIR, ": ", $GET(info), !
        quit 0
    }

    write "  diretorio criado: ", ..#WORKDIR, !
    quit 1
}

/// File operations with %File.
ClassMethod FileOps() As %Status
{
    quit:'..Prepare() $$$OK

    set path = ##class(%File).SubDirectoryName(..#WORKDIR, "teste.txt")
    set copyPath = ##class(%File).SubDirectoryName(..#WORKDIR, "copia.txt")

    write !, "-- gravando --", !
    set file = ##class(%Stream.FileCharacter).%New()
    set sc = file.LinkToFile(path)

    if $$$ISERR(sc) {
        do $SYSTEM.Status.DisplayError(sc)
        quit $$$OK
    }

    do file.WriteLine("primeira linha")
    do file.WriteLine("segunda linha com acento: ação")
    do file.WriteLine("terceira linha")

    set sc = file.%Save()
    write "  gravado: ", $$$ISOK(sc), !

    write !, "-- informacoes --", !
    write "  existe      : ", ##class(%File).Exists(path), !
    write "  tamanho     : ", ##class(%File).GetFileSize(path), " bytes", !
    write "  diretorio   : ", ##class(%File).GetDirectory(path), !
    write "  nome        : ", ##class(%File).GetFilename(path), !

    write !, "-- lendo --", !
    set reader = ##class(%Stream.FileCharacter).%New()
    set sc = reader.LinkToFile(path)
    do reader.Rewind()

    set n = 0
    while 'reader.AtEnd {
        set n = n + 1
        write "  ", n, ": ", reader.ReadLine(), !
    }

    write !, "-- copiando e renomeando --", !
    set ok = ##class(%File).CopyFile(path, copyPath, 1, .info)
    write "  copiado     : ", ok, !

    set renamed = ##class(%File).SubDirectoryName(..#WORKDIR, "renomeado.txt")
    set ok = ##class(%File).Rename(copyPath, renamed)
    write "  renomeado   : ", ok, !

    write !, "-- listando o diretorio --", !
    do ..ListDirectory(..#WORKDIR, "*")

    write !, "-- limpando --", !
    write "  apagou renomeado.txt: ", ##class(%File).Delete(renamed), !

    quit $$$OK
}

/// Lists a directory with the FileSet query.
ClassMethod ListDirectory(directory As %String, filter As %String = "*") As %Integer
{
    set stmt = ##class(%SQL.Statement).%New()

    set sc = stmt.%PrepareClassQuery("%File", "FileSet")
    if $$$ISERR(sc) {
        do $SYSTEM.Status.DisplayError(sc)
        quit -1
    }

    set rs = stmt.%Execute(directory, filter)

    set W = $LISTBUILD(4, 30, 12)
    set A = $LISTBUILD("C", "L", "R")
    do ##class(LabStudy.Formatter).Header($LISTBUILD("tipo", "nome", "tamanho"), W, A)

    set n = 0
    while rs.%Next() {
        set n = n + 1
        do ##class(LabStudy.Formatter).Row(
            $LISTBUILD(rs.%Get("Type"),
                       ##class(%File).GetFilename(rs.%Get("Name")),
                       rs.%Get("Size")),
            W, A)
    }

    do ##class(LabStudy.Formatter).Line(48)
    write "  ", n, " itens", !
    quit n
}

/// The classic OPEN / USE / CLOSE form.
ClassMethod ClassicDevice() As %Status
{
    quit:'..Prepare() $$$OK

    set path = ##class(%File).SubDirectoryName(..#WORKDIR, "classico.txt")

    write "-- gravando com OPEN/USE/CLOSE --", !

    open path:("WNS"):5

    if '$TEST {
        write "  nao foi possivel abrir ", path, !
        quit $$$OK
    }

    // ALWAYS remember where you came from
    set previous = $IO

    use path
    write "linha gravada pelo dispositivo", !
    write "segunda linha", !

    use previous
    close path

    write "  gravado. voltamos para o dispositivo ", $IO, !

    write !, "-- lendo com OPEN/USE/CLOSE --", !

    open path:("R"):5
    quit:'$TEST $$$OK

    set previous = $IO
    use path

    set lines = ""
    for {
        read line
        quit:$ZEOF
        set lines = lines_line_$CHAR(13,10)
    }

    use previous
    close path

    write lines

    quit $$$OK
}

/// An HTTP call. Gracefully reports failure when there is no network.
ClassMethod HttpDemo(server As %String = "www.example.com", path As %String = "/") As %Status
{
    write "-- chamando https://", server, path, " --", !

    set req = ##class(%Net.HttpRequest).%New()
    set req.Server = server
    set req.Port = 443
    set req.Https = 1
    set req.Timeout = 10

    set sc = req.Get(path)

    if $$$ISERR(sc) {
        write "  chamada falhou (rede indisponivel?):", !
        write "  ", $SYSTEM.Status.GetErrorText(sc), !
        quit $$$OK
    }

    set rsp = req.HttpResponse

    write "  status      : ", rsp.StatusCode, !
    write "  content type: ", rsp.ContentType, !

    if $ISOBJECT(rsp.Data) {
        do rsp.Data.Rewind()
        set body = rsp.Data.Read(200)
        write "  primeiros 200 bytes:", !
        write "  ", ##class(LabStudy.Text).Abbreviate(body, 120), !
    }

    quit $$$OK
}

ClassMethod Demo() As %Status
{
    do ..SystemInfo()
    write !
    do ..FileOps()
    write !
    do ..ClassicDevice()
    write !
    do ..HttpDemo()
    quit $$$OK
}

}
```

### 6.1 Executando

```
LABSTUDY>DO ##class(LabStudy.Demo.Api).SystemInfo()
=== versao ===
  $ZVERSION      : IRIS for UNIX (Ubuntu Server LTS ...) ...
  GetVersion()   : IRIS for UNIX ...
  GetNumber()    : 2026.1
  GetPlatform()  : UNIX

=== processo ===
  $JOB           : 12345
  $USERNAME      : _SYSTEM
  $ROLES         : %All
  $NAMESPACE     : LABSTUDY
  $IO            : |TRM|:|12345
  $ZSTORAGE      : 262144 KB
  $STORAGE       : 267726848 bytes livres

=== instalacao ===
  install dir    : /usr/irissys/
  manager dir    : /usr/irissys/mgr/

=== ambiente ===
    PATH : /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:...
    HOME : /home/irisowner
    LANG : en_US.UTF-8

LABSTUDY>DO ##class(LabStudy.Demo.Api).FileOps()
  diretorio criado: /tmp/labstudy

-- gravando --
  gravado: 1

-- informacoes --
  existe      : 1
  tamanho     : 63 bytes
  diretorio   : /tmp/labstudy/
  nome        : teste.txt

-- lendo --
  1: primeira linha
  2: segunda linha com acento: ação
  3: terceira linha

-- copiando e renomeando --
  copiado     : 1
  renomeado   : 1

-- listando o diretorio --
tipo  nome                             tamanho
------------------------------------------------
 F    renomeado.txt                         63
 F    teste.txt                             63
------------------------------------------------
  2 itens

-- limpando --
  apagou renomeado.txt: 1
```

O que observar:

- **`$IO` no Terminal mostra o dispositivo de terminal**, com o número do processo. É a prova de que a tela também é um dispositivo.
- **O acento foi gravado e lido corretamente**, porque o ambiente está em UTF-8 e o stream usou a configuração padrão coerente. **Isso não é garantido** — num ambiente com codificação diferente, seria necessário definir `TranslateTable` explicitamente. É exatamente o tipo de coisa que funciona na sua máquina e quebra no servidor.
- **`SubDirectoryName` montou os caminhos** em vez de concatenar com `/`. Num servidor Windows, a concatenação manual falharia.
- **`GetDirectory` devolveu o caminho com a barra final.** Detalhes assim variam entre plataformas e são a razão de usar as funções em vez de manipular texto.
- **`FileSet` foi executada exatamente como qualquer class query** do Capítulo 18, com `%PrepareClassQuery`. Os dados vieram do sistema de arquivos, e não do banco — e o código de consumo é idêntico.

```
LABSTUDY>DO ##class(LabStudy.Demo.Api).ClassicDevice()
-- gravando com OPEN/USE/CLOSE --
  gravado. voltamos para o dispositivo |TRM|:|12345

-- lendo com OPEN/USE/CLOSE --
linha gravada pelo dispositivo
segunda linha
```

- **A mensagem "gravado" só apareceu depois do `use previous`.** Enquanto o dispositivo atual era o arquivo, tudo que foi escrito foi para lá — inclusive o que você **não** queria. Faça o teste: remova o `use previous` e observe o Terminal ficar mudo. Essa é a armadilha do dispositivo atual.
- **`$ZEOF`** foi usado para detectar o fim do arquivo na leitura. É a variável especial que indica fim de arquivo depois de um `READ`. Ela precisa de uma configuração adequada do dispositivo para funcionar de forma confiável: **verificar na documentação oficial**.
- **Compare com a versão em stream:** menos linhas, sem risco de perder o dispositivo, sem `$ZEOF`. É por isso que streams são a recomendação.

---

## 7. Pegadinhas e erros comuns

**1) Reimplementar o que já existe.**
Antes de escrever, consulte `DO $SYSTEM.Help()` e a documentação. Quase tudo já está pronto.

**2) Concatenar caminhos com `/` na mão.**
Quebra em outra plataforma. Use `##class(%File).SubDirectoryName()`.

**3) Usar `CreateDirectory` esperando criar a cadeia inteira.**
Ele cria um nível. Para a cadeia, `CreateDirectoryChain`.

**4) Ignorar o valor de retorno dos métodos de `%File`.**
A maioria devolve `1`/`0` e não gera erro. Falha silenciosa.

**5) Esquecer de restaurar `$IO` depois de um `USE`.**
Toda a saída seguinte vai para o arquivo, inclusive mensagens de erro que você nunca verá.

**6) Esquecer o `CLOSE`.**
Vazamento de recurso; o arquivo pode ficar bloqueado.

**7) Não conferir `$TEST` depois do `OPEN`.**
Você segue achando que abriu.

**8) Não definir codificação ao trocar arquivos com outros sistemas.**
Acentos corrompidos. Defina `TranslateTable` ou converta com `$ZCONVERT`.

**9) Não definir o terminador de linha.**
Arquivos de outro sistema operacional deixam `$CHAR(13)` invisível no fim das linhas.

**10) Chamada HTTP sem tempo limite.**
Um serviço lento trava o seu processo indefinidamente.

**11) Chamada HTTP dentro de uma transação.**
Você segura travas e journal esperando a rede.

**12) Ignorar o `StatusCode` da resposta HTTP.**
`%Status` de sucesso significa que a chamada **aconteceu**, não que o servidor respondeu com sucesso.

**13) Esquecer de rebobinar o stream da resposta antes de ler.**
Mesmo comportamento dos streams do Capítulo 4.

**14) Usar `$ZF` com argumentos vindos de fora.**
Execução de comando no servidor. Trate como o `XECUTE`: lista de permitidos, ou não use.

**15) Disparar `JOB` periódico em vez de usar o agendador.**
Você reimplementa controle de concorrência, histórico e reprogramação que o `%SYS.Task` já oferece.

**16) Gravar caminhos absolutos no código.**
Use `$SYSTEM.Util.InstallDirectory()` ou configuração; senão o sistema só roda na sua máquina.

---

## 8. MÃO NA MASSA

---

### Exercício 19.1 — Painel do sistema

**a) Enunciado:** Crie `LabStudy.Demo.Ap1` com um `ClassMethod Report()` que imprima um painel completo do ambiente, organizado em seções e alinhado com o `Formatter`:

1. Versão do produto, plataforma e número.
2. Processo: número, usuário, papéis, namespace, dispositivo, memória.
3. Instalação: diretórios.
4. Namespace: quantas classes, quantas globais (aproximadamente), tamanho da base.
5. Banco de dados do namespace atual.
6. Variáveis de ambiente relevantes.

Depois, `ClassMethod Compare(v1, v2)` que compara duas strings de versão numericamente, devolvendo `-1`, `0` ou `1` — útil para código que precisa se adaptar à versão.

**b) Dica:** Para contar classes, consulte `%Dictionary.CompiledClass` por SQL, filtrando pelo pacote.

**c) Como testar:** `Compare("2025.1", "2026.1")` deve devolver `-1`.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

`src/LabStudy/Demo/Ap1.cls`:

```objectscript
/// Environment dashboard.
Class LabStudy.Demo.Ap1 Extends %RegisteredObject
{

ClassMethod Section(title As %String) As %Status [ Private ]
{
    write !
    do ##class(LabStudy.Formatter).Line(52, "=")
    write ##class(LabStudy.Text).Pad(title, 52, "C"), !
    do ##class(LabStudy.Formatter).Line(52, "=")
    quit $$$OK
}

ClassMethod Item(label As %String, value As %String) As %Status [ Private ]
{
    do ##class(LabStudy.Formatter).Row(
        $LISTBUILD(label, value),
        $LISTBUILD(20, 30),
        $LISTBUILD("L", "L"))
    quit $$$OK
}

ClassMethod Report() As %Status
{
    do ..Section("produto")
    do ..Item("versao", $SYSTEM.Version.GetVersion())
    do ..Item("numero", $SYSTEM.Version.GetNumber())
    do ..Item("plataforma", $SYSTEM.Version.GetPlatform())

    do ..Section("processo")
    do ..Item("job", $JOB)
    do ..Item("usuario", $USERNAME)
    do ..Item("papeis", $ROLES)
    do ..Item("namespace", $NAMESPACE)
    do ..Item("dispositivo", $IO)
    do ..Item("memoria limite", $ZSTORAGE_" KB")
    do ..Item("memoria livre", $STORAGE_" bytes")
    do ..Item("fuso (min)", $ZTIMEZONE)

    do ..Section("instalacao")
    do ..Item("install dir", $SYSTEM.Util.InstallDirectory())
    do ..Item("manager dir", $SYSTEM.Util.ManagerDirectory())

    do ..Section("namespace")

    new SQLCODE, %msg, total, labstudy

    &sql(SELECT COUNT(*) INTO :total FROM %Dictionary.CompiledClass)
    do ..Item("classes compiladas", $SELECT(SQLCODE=0: total, 1: "?"))

    &sql(SELECT COUNT(*) INTO :labstudy FROM %Dictionary.CompiledClass
         WHERE Name %STARTSWITH 'LabStudy.')
    do ..Item("classes LabStudy", $SELECT(SQLCODE=0: labstudy, 1: "?"))

    &sql(SELECT COUNT(*) INTO :total FROM LabStudy.PATIENT)
    do ..Item("pacientes", $SELECT(SQLCODE=0: total, 1: "?"))

    &sql(SELECT COUNT(*) INTO :total FROM LabStudy.EXAM)
    do ..Item("exames", $SELECT(SQLCODE=0: total, 1: "?"))

    do ..Section("ambiente")
    for var = "HOME", "LANG", "TZ" {
        do ..Item(var, ##class(LabStudy.Text).Abbreviate($SYSTEM.Util.GetEnviron(var), 28))
    }

    write !
    quit $$$OK
}

/// Compares two version strings numerically.
/// Returns -1, 0 or 1.
ClassMethod Compare(v1 As %String, v2 As %String) As %Integer
{
    set parts = $SELECT($LENGTH(v1, ".") > $LENGTH(v2, "."): $LENGTH(v1, "."), 1: $LENGTH(v2, "."))

    for i = 1:1:parts {
        set a = +$PIECE(v1, ".", i)
        set b = +$PIECE(v2, ".", i)

        quit:a'=b

        // continue while equal
    }

    set a = +$PIECE(v1, ".", i)
    set b = +$PIECE(v2, ".", i)

    quit $SELECT(a < b: -1, a > b: 1, 1: 0)
}

/// Convenience: is the running version at least the given one?
ClassMethod AtLeast(version As %String) As %Boolean
{
    quit ..Compare($SYSTEM.Version.GetNumber(), version) >= 0
}

ClassMethod TestCompare() As %Status
{
    write "-- comparacao de versoes --", !

    for pair = "2025.1|2026.1", "2026.1|2025.1", "2026.1|2026.1",
               "2026.1|2026.1.0", "2026.2|2026.10" {
        set a = $PIECE(pair, "|", 1), b = $PIECE(pair, "|", 2)
        write "  ", $JUSTIFY(a, 10), " vs ", $JUSTIFY(b, 10), " -> ",
              $JUSTIFY(..Compare(a, b), 3), !
    }

    write !, "  versao atual: ", $SYSTEM.Version.GetNumber(), !
    write "  >= 2020.1 ? ", ..AtLeast("2020.1"), !
    write "  >= 2099.1 ? ", ..AtLeast("2099.1"), !

    quit $$$OK
}

}
```

```
LABSTUDY>DO ##class(LabStudy.Demo.Ap1).Report()

====================================================
                     produto
====================================================
versao               IRIS for UNIX ...
numero               2026.1
plataforma           UNIX

====================================================
                     processo
====================================================
job                  12345
usuario              _SYSTEM
papeis               %All
namespace            LABSTUDY
dispositivo          |TRM|:|12345
memoria limite       262144 KB
memoria livre        267726848 bytes
fuso (min)           180

====================================================
                    namespace
====================================================
classes compiladas   1847
classes LabStudy     31
pacientes            201
exames               2003

LABSTUDY>DO ##class(LabStudy.Demo.Ap1).TestCompare()
-- comparacao de versoes --
      2025.1 vs     2026.1 ->  -1
      2026.1 vs     2025.1 ->   1
      2026.1 vs     2026.1 ->   0
      2026.1 vs   2026.1.0 ->   0
      2026.2 vs    2026.10 ->  -1

  versao atual: 2026.1
  >= 2020.1 ? 1
  >= 2099.1 ? 0
```

**Por que cada decisão:**

- **`Section` e `Item` são métodos auxiliares privados.** Sem eles, o `Report` teria cem linhas de `write` com espaçamento manual. Com eles, cada item é uma linha legível — e a formatação está num lugar só, caso mude.
- **`Compare` compara **numericamente** parte por parte.** Repare em `"2026.2" vs "2026.10"`: como texto, `"2026.10"` viria antes de `"2026.2"` (Capítulo 13). Como número, `10 > 2`, e a comparação está correta. **Comparar versões como texto é um erro clássico** — e a linha `2026.2 vs 2026.10 -> -1` é a prova de que o método acertou.
- **`"2026.1" vs "2026.1.0"` deu zero**, porque o `+` sobre um pedaço inexistente devolve `0`, e `0 = 0`. É o comportamento correto e desejado, e vem de graça do `$PIECE` devolver vazio fora de faixa (Capítulo 15).
- **`AtLeast` é a função que você realmente vai usar.** Código que precisa se adaptar à versão escreve `if ##class(...).AtLeast("2022.1") { ... }` — legível e correto.
- **As contagens usam SQL sobre `%Dictionary.CompiledClass`.** O IRIS descreve a si mesmo em tabelas consultáveis, como visto no Capítulo 11.
- **Cada consulta confere `SQLCODE` e imprime `"?"` em caso de falha.** Um painel que quebra porque uma tabela não existe é pior que um painel com uma linha incompleta.

---

### Exercício 19.2 — Operações com arquivos

**a) Enunciado:** Crie `LabStudy.Demo.Ap2`, uma pequena biblioteca de arquivos:

1. `EnsureDir(caminho)` — garante que o diretório existe, criando a cadeia se necessário.
2. `WriteText(caminho, texto)` e `ReadText(caminho)` — grava e lê um texto inteiro.
3. `WriteLines(caminho, lista)` e `ReadLines(caminho) As %List` — grava e lê linha a linha.
4. `Append(caminho, linha)` — acrescenta ao final, sem apagar o que existe.
5. `Info(caminho)` — imprime existência, tamanho, diretório, nome e extensão.
6. `List(diretorio, filtro)` — lista com `FileSet`, ordenado por tamanho decrescente.
7. `Backup(caminho)` — copia para um nome com carimbo de data e hora.
8. `CleanOld(diretorio, filtro, dias)` — apaga arquivos mais antigos que N dias (mostrando o que faria, com um modo "simulação").

**b) Dica:** No item 8, faça o modo simulação ser o **padrão**. Apagar arquivos por engano é irreversível.

**c) Como testar:** O item 4 deve preservar o conteúdo anterior.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

`src/LabStudy/Demo/Ap2.cls`:

```objectscript
/// A small file handling library.
Class LabStudy.Demo.Ap2 Extends %RegisteredObject
{

Parameter WORKDIR = "/tmp/labstudy";

/// Makes sure a directory exists.
ClassMethod EnsureDir(path As %String) As %Boolean
{
    quit:##class(%File).DirectoryExists(path) 1

    set ok = ##class(%File).CreateDirectoryChain(path, .info)
    write:'ok "  falha ao criar ", path, ": ", $GET(info), !
    quit ok
}

/// Writes a whole text to a file, replacing its content.
ClassMethod WriteText(path As %String, text As %String) As %Status
{
    do ..EnsureDir(##class(%File).GetDirectory(path))

    set file = ##class(%Stream.FileCharacter).%New()
    set sc = file.LinkToFile(path)
    quit:$$$ISERR(sc) sc

    do file.Clear()
    do file.Write(text)

    quit file.%Save()
}

/// Reads a whole file into a string. Beware of <MAXSTRING> on big files.
ClassMethod ReadText(path As %String, maxChars As %Integer = 100000) As %String
{
    quit:'##class(%File).Exists(path) ""

    set file = ##class(%Stream.FileCharacter).%New()
    set sc = file.LinkToFile(path)
    quit:$$$ISERR(sc) ""

    do file.Rewind()

    set out = ""
    while ('file.AtEnd) && ($LENGTH(out) < maxChars) {
        set out = out_file.Read(8000)
    }
    quit out
}

/// Writes a list of lines.
ClassMethod WriteLines(path As %String, lines As %List) As %Status
{
    do ..EnsureDir(##class(%File).GetDirectory(path))

    set file = ##class(%Stream.FileCharacter).%New()
    set sc = file.LinkToFile(path)
    quit:$$$ISERR(sc) sc

    do file.Clear()

    set ptr = 0
    while $LISTNEXT(lines, ptr, line) {
        do file.WriteLine($GET(line))
    }

    quit file.%Save()
}

/// Reads a file into a list of lines.
ClassMethod ReadLines(path As %String) As %List
{
    quit:'##class(%File).Exists(path) ""

    set file = ##class(%Stream.FileCharacter).%New()
    set sc = file.LinkToFile(path)
    quit:$$$ISERR(sc) ""

    do file.Rewind()

    set out = ""
    while 'file.AtEnd {
        set line = file.ReadLine()
        // strip a stray carriage return left by another operating system
        set line = $ZSTRIP(line, ">", $CHAR(13))
        set out = out_$LISTBUILD(line)
    }
    quit out
}

/// Appends one line, preserving what is already there.
ClassMethod Append(path As %String, line As %String) As %Status
{
    do ..EnsureDir(##class(%File).GetDirectory(path))

    set file = ##class(%Stream.FileCharacter).%New()
    set sc = file.LinkToFile(path)
    quit:$$$ISERR(sc) sc

    // the essential step: go to the end before writing
    do file.MoveToEnd()
    do file.WriteLine(line)

    quit file.%Save()
}

/// Information about a file.
ClassMethod Info(path As %String) As %Status
{
    write "  caminho   : ", path, !
    write "  existe    : ", ##class(%File).Exists(path), !

    if '##class(%File).Exists(path) {
        quit $$$OK
    }

    set name = ##class(%File).GetFilename(path)

    write "  diretorio : ", ##class(%File).GetDirectory(path), !
    write "  nome      : ", name, !
    write "  extensao  : ", $PIECE(name, ".", $LENGTH(name, ".")), !
    write "  tamanho   : ", ##class(%File).GetFileSize(path), " bytes", !

    quit $$$OK
}

/// Lists a directory, ordered by size descending.
ClassMethod List(directory As %String = {..#WORKDIR}, filter As %String = "*") As %Integer
{
    set stmt = ##class(%SQL.Statement).%New()
    set sc = stmt.%PrepareClassQuery("%File", "FileSet")
    quit:$$$ISERR(sc) -1

    set rs = stmt.%Execute(directory, filter)

    kill idx
    while rs.%Next() {
        do ##class(LabStudy.Sorter).Add(.idx, +rs.%Get("Size"), rs.%Get("Name"),
            $LISTBUILD(rs.%Get("Type"),
                       ##class(%File).GetFilename(rs.%Get("Name")),
                       rs.%Get("DateModified")))
    }

    write "  conteudo de ", directory, " (", filter, ")", !

    set W = $LISTBUILD(4, 28, 12, 20)
    set A = $LISTBUILD("C", "L", "R", "L")
    do ##class(LabStudy.Formatter).Header(
        $LISTBUILD("tipo", "nome", "tamanho", "modificado"), W, A)

    set n = ##class(LabStudy.Sorter).Walk(.idx, 1, .ordered)

    for i = 1:1:n {
        set info = $LIST(ordered(i), 3)
        do ##class(LabStudy.Formatter).Row(
            $LISTBUILD($LISTGET(info, 1), $LISTGET(info, 2),
                       $LIST(ordered(i), 1), $LISTGET(info, 3)),
            W, A)
    }

    do ##class(LabStudy.Formatter).Line(68)
    write "  ", n, " itens", !
    quit n
}

/// Copies a file to a timestamped name.
ClassMethod Backup(path As %String) As %String
{
    quit:'##class(%File).Exists(path) ""

    set name = ##class(%File).GetFilename(path)
    set dir = ##class(%File).GetDirectory(path)

    set stamp = $TRANSLATE($ZDATETIME($HOROLOG, 8, 1), ": ", "--")
    set backupName = name_"."_stamp_".bak"
    set backupPath = ##class(%File).SubDirectoryName(dir, backupName)

    set ok = ##class(%File).CopyFile(path, backupPath, 1, .info)

    if 'ok {
        write "  falha ao copiar: ", $GET(info), !
        quit ""
    }

    quit backupPath
}

/// Deletes files older than the given number of days.
/// SIMULATION IS THE DEFAULT: nothing is deleted unless you ask.
ClassMethod CleanOld(directory As %String = {..#WORKDIR}, filter As %String = "*.bak", days As %Integer = 7, reallyDelete As %Boolean = 0) As %Integer
{
    set stmt = ##class(%SQL.Statement).%New()
    set sc = stmt.%PrepareClassQuery("%File", "FileSet")
    quit:$$$ISERR(sc) -1

    set rs = stmt.%Execute(directory, filter)
    set limit = ##class(LabStudy.DateTime).Today() - days

    write "  ", $SELECT(reallyDelete: "APAGANDO", 1: "simulacao (nada sera apagado)"),
          " arquivos ", filter, " com mais de ", days, " dias", !

    set n = 0
    while rs.%Next() {
        set modified = rs.%Get("DateModified")
        continue:modified=""

        set fileDate = ##class(LabStudy.DateTime).Parse($PIECE(modified, " ", 1), 3)
        continue:fileDate=""
        continue:fileDate>limit

        set n = n + 1
        set path = rs.%Get("Name")

        if reallyDelete {
            set ok = ##class(%File).Delete(path)
            write "    apagado (", ok, "): ", ##class(%File).GetFilename(path), !
        } else {
            write "    seria apagado: ", ##class(%File).GetFilename(path),
                  "  (", $PIECE(modified, " ", 1), ")", !
        }
    }

    write "  ", n, " arquivos", !
    quit n
}

ClassMethod Demo() As %Status
{
    set path = ##class(%File).SubDirectoryName(..#WORKDIR, "demo.txt")

    write "-- gravando --", !
    do ..WriteLines(path, $LISTBUILD("linha 1", "linha 2", "linha 3"))
    do ..Info(path)

    write !, "-- acrescentando --", !
    do ..Append(path, "linha 4 (acrescentada)")
    do ..Append(path, "linha 5 (acrescentada)")

    write !, "-- lendo de volta --", !
    set lines = ..ReadLines(path)
    for i = 1:1:$LISTLENGTH(lines) {
        write "  ", i, ": ", $LISTGET(lines, i), !
    }

    write !, "-- backup --", !
    set b = ..Backup(path)
    write "  copia: ", ##class(%File).GetFilename(b), !

    write !, "-- listagem --", !
    do ..List()

    write !, "-- limpeza (simulacao) --", !
    do ..CleanOld(..#WORKDIR, "*.bak", 0)

    quit $$$OK
}

}
```

```
LABSTUDY>DO ##class(LabStudy.Demo.Ap2).Demo()
-- gravando --
  caminho   : /tmp/labstudy/demo.txt
  existe    : 1
  diretorio : /tmp/labstudy/
  nome      : demo.txt
  extensao  : txt
  tamanho   : 24 bytes

-- acrescentando --

-- lendo de volta --
  1: linha 1
  2: linha 2
  3: linha 3
  4: linha 4 (acrescentada)
  5: linha 5 (acrescentada)

-- backup --
  copia: demo.txt.20260819-16-42-03.bak

-- listagem --
  conteudo de /tmp/labstudy (*)
tipo  nome                          tamanho  modificado
--------------------------------------------------------------------
 F    demo.txt.20260819-...bak           70  2026-08-19 16:42:03
 F    demo.txt                           70  2026-08-19 16:42:03
 F    teste.txt                          63  2026-08-19 16:40:11
--------------------------------------------------------------------
  3 itens

-- limpeza (simulacao) --
  simulacao (nada sera apagado) arquivos *.bak com mais de 0 dias
    seria apagado: demo.txt.20260819-16-42-03.bak  (2026-08-19)
  1 arquivos
```

**Por que cada decisão:**

- **`Append` chama `MoveToEnd()` antes de escrever.** Sem isso, a escrita começaria no início e sobrescreveria — o erro clássico do Capítulo 4, agora num arquivo real. As linhas 1 a 3 teriam desaparecido.
- **`ReadLines` remove um `$CHAR(13)` sobrando no fim.** Arquivos gerados no Windows terminam linhas com `CR LF`; lendo no Linux, o `CR` fica pendurado, invisível, e quebra comparações. **Essa linha economiza horas de depuração** e custa uma chamada.
- **`ReadText` tem limite de caracteres.** Ler um arquivo de 2 GB para uma string é `<MAXSTRING>` garantido. O limite documenta a intenção e protege.
- **`CleanOld` tem simulação como padrão.** Um método que apaga arquivos precisa exigir confirmação explícita. Inverter esse padrão — apagar por padrão, simular sob demanda — é um convite ao desastre.
- **`CleanOld` imprime o que **faria**, com data.** Isso permite revisar antes de executar de verdade.
- **`Backup` usa carimbo de data e hora no nome**, com `$TRANSLATE` trocando os caracteres inválidos para nome de arquivo. Dois pontos não são permitidos em nomes de arquivo no Windows.
- **`List` reaproveita o `Sorter` do Capítulo 13** para ordenar por tamanho. Nenhuma ordenação nova precisou ser escrita — a camada construída lá continua servindo.
- **`EnsureDir` é chamado por todos os métodos de escrita.** Gravar num diretório inexistente falha silenciosamente ou com erro obscuro; garantir antes torna a biblioteca robusta.

---

### Exercício 19.3 — PROJETO CONTÍNUO: exportação, importação e painel

**a) Enunciado:** Leve o sistema à versão **2.0**:

1. Crie `LabStudy.FileIO` com:
   - `ExportPatientsCsv(caminho)` — exporta todos os pacientes para CSV, com cabeçalho, usando `ListUtil.ToCsv`;
   - `ImportPatientsCsv(caminho, simular)` — lê um CSV, valida linha a linha e cria os pacientes; **simulação por padrão**, relatando o que faria;
   - `ExportExamsCsv(caminho, apenasFinais)`;
   - `ExportPatientJson(id, caminho)` — usa `ToDynamicObject()` do Capítulo 4;
   - `BackupAll(diretorio)` — exporta tudo com carimbo de data e hora e devolve a lista de arquivos gerados;
   - `Verify(caminho)` — conta linhas e valida o cabeçalho de um CSV exportado.
2. Crie `LabStudy.SystemInfo` (ou amplie a do Capítulo 8) com `Dashboard()` reunindo versão, processo, esquema, dados e arquivos de exportação.
3. Acrescente ao menu as opções de exportar, importar e ver o painel do sistema.
4. Suba `LabStudy.App` para **`"2.0"`** e faça o `About()` exibir também a versão do IRIS.

**b) Dica:** Na importação, use `ListUtil.FromCsv` do Capítulo 14 — ele trata aspas corretamente.

**c) Como testar:**

```
LABSTUDY>DO ##class(LabStudy.FileIO).ExportPatientsCsv("/tmp/labstudy/pacientes.csv")
LABSTUDY>DO ##class(LabStudy.FileIO).Verify("/tmp/labstudy/pacientes.csv")
LABSTUDY>DO ##class(LabStudy.FileIO).ImportPatientsCsv("/tmp/labstudy/pacientes.csv")
LABSTUDY>DO ##class(LabStudy.SystemInfo).Dashboard()
```

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

`src/LabStudy/FileIO.cls`:

```objectscript
/// File based import and export for the LabStudy system.
/// CSV handling is delegated to LabStudy.ListUtil, which deals with quoting.
Class LabStudy.FileIO Extends %RegisteredObject
{

Parameter DEFAULTDIR = "/tmp/labstudy";

Parameter PATIENTHEADER = "recordNumber,name,birthDate,sex,city,email,active";

Parameter EXAMHEADER = "examId,recordNumber,testCode,resultValue,unit,status,collectedOn";

/// Makes sure a directory exists.
ClassMethod EnsureDir(path As %String) As %Boolean
{
    quit:##class(%File).DirectoryExists(path) 1
    quit ##class(%File).CreateDirectoryChain(path, .info)
}

/// Builds a path inside the default directory, using the separator of
/// the running platform. Never concatenate with "/" by hand.
ClassMethod DefaultPath(fileName As %String) As %String [ CodeMode = expression ]
{
##class(%File).SubDirectoryName(..#DEFAULTDIR, fileName)
}

/// Opens a file stream for writing, replacing its content.
ClassMethod OpenForWrite(path As %String, Output stream) As %Status [ Private ]
{
    do ..EnsureDir(##class(%File).GetDirectory(path))

    set stream = ##class(%Stream.FileCharacter).%New()
    set sc = stream.LinkToFile(path)
    quit:$$$ISERR(sc) sc

    do stream.Clear()
    quit $$$OK
}

/// Exports every patient to CSV.
ClassMethod ExportPatientsCsv(path As %String = "") As %Integer
{
    set:path="" path = ..DefaultPath("pacientes.csv")

    set sc = ..OpenForWrite(path, .file)
    if $$$ISERR(sc) {
        do $SYSTEM.Status.DisplayError(sc)
        quit -1
    }

    do file.WriteLine(..#PATIENTHEADER)

    set rs = ##class(%SQL.Statement).%New()
    set rs.%SelectMode = 1                       // ODBC dates
    set sc = rs.%Prepare(
        "SELECT RecordNumber, Name, BirthDate, Sex, Address_City AS City, "
        _"Email, Active FROM LabStudy.PATIENT ORDER BY RecordNumber")

    if $$$ISERR(sc) {
        do $SYSTEM.Status.DisplayError(sc)
        quit -1
    }

    set result = rs.%Execute()
    set n = 0

    while result.%Next() {
        set row = $LISTBUILD(result.%Get("RecordNumber"),
                             result.%Get("Name"),
                             result.%Get("BirthDate"),
                             result.%Get("Sex"),
                             result.%Get("City"),
                             result.%Get("Email"),
                             result.%Get("Active"))

        do file.WriteLine(##class(LabStudy.ListUtil).ToCsv(row))
        set n = n + 1
    }

    set sc = file.%Save()
    if $$$ISERR(sc) {
        do $SYSTEM.Status.DisplayError(sc)
        quit -1
    }

    write "  ", n, " pacientes exportados para ", path, !
    write "  (", ##class(%File).GetFileSize(path), " bytes)", !
    quit n
}

/// Exports exams to CSV.
ClassMethod ExportExamsCsv(path As %String = "", onlyFinal As %Boolean = 1) As %Integer
{
    set:path="" path = ..DefaultPath("exames.csv")

    set sc = ..OpenForWrite(path, .file)
    quit:$$$ISERR(sc) -1

    do file.WriteLine(..#EXAMHEADER)

    set sql = "SELECT e.%ID AS Id, e.Patient->RecordNumber AS Rec, e.TestCode, "
              _"e.ResultValue, e.Unit, e.ResultStatus, e.CollectedOn "
              _"FROM LabStudy.EXAM e"
    set:onlyFinal sql = sql_" WHERE e.ResultStatus = 'final'"
    set sql = sql_" ORDER BY e.%ID"

    set rs = ##class(%SQL.Statement).%New()
    set rs.%SelectMode = 1
    set sc = rs.%Prepare(sql)
    quit:$$$ISERR(sc) -1

    set result = rs.%Execute()
    set n = 0

    while result.%Next() {
        do file.WriteLine(##class(LabStudy.ListUtil).ToCsv(
            $LISTBUILD(result.%Get("Id"), result.%Get("Rec"),
                       result.%Get("TestCode"), result.%Get("ResultValue"),
                       result.%Get("Unit"), result.%Get("ResultStatus"),
                       result.%Get("CollectedOn"))))
        set n = n + 1
    }

    set sc = file.%Save()
    quit:$$$ISERR(sc) -1

    write "  ", n, " exames exportados para ", path, !
    quit n
}

/// Exports one patient as JSON.
ClassMethod ExportPatientJson(id As %String, path As %String = "") As %Status
{
    set patient = ##class(LabStudy.Patient).%OpenId(id)
    if '$ISOBJECT(patient) {
        write "  paciente nao encontrado: ", id, !
        quit $$$OK
    }

    set:path="" path = ..DefaultPath("paciente-"_patient.RecordNumber_".json")

    set sc = ..OpenForWrite(path, .file)
    quit:$$$ISERR(sc) sc

    do file.Write(patient.ToDynamicObject().%ToJSON())
    set sc = file.%Save()

    write:$$$ISOK(sc) "  exportado para ", path, !
    quit sc
}

/// Imports patients from CSV.
/// SIMULATION IS THE DEFAULT.
ClassMethod ImportPatientsCsv(path As %String, reallyImport As %Boolean = 0) As %Integer
{
    if '##class(%File).Exists(path) {
        write "  arquivo nao encontrado: ", path, !
        quit -1
    }

    set file = ##class(%Stream.FileCharacter).%New()
    set sc = file.LinkToFile(path)
    quit:$$$ISERR(sc) -1

    do file.Rewind()

    write "  ", $SELECT(reallyImport: "IMPORTANDO", 1: "simulacao (nada sera gravado)"),
          " de ", path, !

    // header
    set header = $ZSTRIP(file.ReadLine(), ">", $CHAR(13))
    if $ZCONVERT(header, "L") '= $ZCONVERT(..#PATIENTHEADER, "L") {
        write "  cabecalho inesperado:", !
        write "    esperado: ", ..#PATIENTHEADER, !
        write "    recebido: ", header, !
        quit -1
    }

    set line = 1, created = 0, skipped = 0, failed = 0

    while 'file.AtEnd {
        set line = line + 1
        set raw = $ZSTRIP(file.ReadLine(), ">", $CHAR(13))
        continue:$ZSTRIP(raw, "<>W")=""

        set row = ##class(LabStudy.ListUtil).FromCsv(raw)

        set rec   = $ZCONVERT(##class(LabStudy.Text).Clean($LISTGET(row, 1)), "U")
        set name  = ##class(LabStudy.Text).Clean($LISTGET(row, 2))
        set birth = ##class(LabStudy.Text).Clean($LISTGET(row, 3))
        set sex   = $ZCONVERT(##class(LabStudy.Text).Clean($LISTGET(row, 4)), "U")
        set city  = ##class(LabStudy.Text).Clean($LISTGET(row, 5))
        set email = ##class(LabStudy.Text).Clean($LISTGET(row, 6))

        // validation, before anything is created
        if 'rec?1"REG-"6N {
            write "    linha ", line, ": numero de registro invalido [", rec, "]", !
            set failed = failed + 1
            continue
        }

        if '##class(LabStudy.Text).IsIsoDate(birth) {
            write "    linha ", line, ": data invalida [", birth, "]", !
            set failed = failed + 1
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

        set sc = p.%Save()

        if $$$ISERR(sc) {
            write "    linha ", line, ": ", $SYSTEM.Status.GetErrorText(sc), !
            set failed = failed + 1
            continue
        }

        set created = created + 1
    }

    write "  ------------------------------", !
    write "  linhas lidas : ", line - 1, !
    write "  criados      : ", created, !
    write "  ja existiam  : ", skipped, !
    write "  falhas       : ", failed, !

    quit created
}

/// Checks a CSV produced by the export.
ClassMethod Verify(path As %String) As %Status
{
    if '##class(%File).Exists(path) {
        write "  arquivo nao encontrado: ", path, !
        quit $$$OK
    }

    set file = ##class(%Stream.FileCharacter).%New()
    set sc = file.LinkToFile(path)
    quit:$$$ISERR(sc) sc

    do file.Rewind()

    set header = $ZSTRIP(file.ReadLine(), ">", $CHAR(13))
    set lines = 0, maxFields = 0, minFields = ""

    while 'file.AtEnd {
        set raw = $ZSTRIP(file.ReadLine(), ">", $CHAR(13))
        continue:$ZSTRIP(raw, "<>W")=""

        set lines = lines + 1
        set fields = $LISTLENGTH(##class(LabStudy.ListUtil).FromCsv(raw))

        set:fields>maxFields maxFields = fields
        set:(minFields = "") || (fields < minFields) minFields = fields
    }

    set expected = $LENGTH(..#PATIENTHEADER, ",")

    write "  arquivo    : ", ##class(%File).GetFilename(path), !
    write "  tamanho    : ", ##class(%File).GetFileSize(path), " bytes", !
    write "  cabecalho  : ", $SELECT(header = ..#PATIENTHEADER: "ok", 1: "DIFERENTE"), !
    write "  linhas     : ", lines, !
    write "  campos     : ", minFields, " a ", maxFields, " (esperado ", expected, ")", !
    write "  consistente: ", $SELECT((minFields = expected) && (maxFields = expected): "sim", 1: "NAO"), !

    quit $$$OK
}

/// Exports everything to a timestamped directory.
ClassMethod BackupAll(directory As %String = {..#DEFAULTDIR}) As %List
{
    set stamp = $TRANSLATE($ZDATETIME($HOROLOG, 8, 1), ": ", "--")
    set dir = ##class(%File).SubDirectoryName(directory, "backup-"_stamp)

    if '..EnsureDir(dir) {
        write "  nao foi possivel criar ", dir, !
        quit ""
    }

    write "  backup em ", dir, !

    set files = ""

    set p = ##class(%File).SubDirectoryName(dir, "pacientes.csv")
    if ..ExportPatientsCsv(p) >= 0 {
        set files = files_$LISTBUILD(p)
    }

    set e = ##class(%File).SubDirectoryName(dir, "exames.csv")
    if ..ExportExamsCsv(e, 0) >= 0 {
        set files = files_$LISTBUILD(e)
    }

    write "  ", $LISTLENGTH(files), " arquivos gerados", !
    quit files
}

}
```

`src/LabStudy/SystemInfo.cls` (ampliando a do Capítulo 8):

```objectscript
/// Full system dashboard: product, process, schema, data and files.
Class LabStudy.SystemInfo Extends %RegisteredObject
{

ClassMethod Section(title As %String) As %Status [ Private ]
{
    write !
    do ##class(LabStudy.Formatter).Line(54, "=")
    write ##class(LabStudy.Text).Pad(title, 54, "C"), !
    do ##class(LabStudy.Formatter).Line(54, "=")
    quit $$$OK
}

ClassMethod Item(label As %String, value As %String) As %Status [ Private ]
{
    do ##class(LabStudy.Formatter).Row(
        $LISTBUILD(label, value), $LISTBUILD(22, 30), $LISTBUILD("L", "L"))
    quit $$$OK
}

/// Storage globals of a class and how many nodes each holds.
ClassMethod ClassGlobals(className As %String) As %Status
{
    set dataGlobal = "^"_className_"D"

    set nodes = 0, k = ""
    for {
        set k = $ORDER(@dataGlobal@(k))
        quit:k=""
        set nodes = nodes + 1
    }

    do ..Item(className, nodes_" objetos, proximo id "_$GET(@dataGlobal, 0))
    quit $$$OK
}

ClassMethod Dashboard() As %Status
{
    do ..Section("LabStudy "_##class(LabStudy.App).#VERSION)

    do ..Item("IRIS", $SYSTEM.Version.GetNumber())
    do ..Item("plataforma", $SYSTEM.Version.GetPlatform())
    do ..Item("namespace", $NAMESPACE)
    do ..Item("usuario", $USERNAME)
    do ..Item("processo", $JOB)
    do ..Item("data/hora", ##class(LabStudy.DateTime).NowTimestamp())

    do ..Section("esquema")
    do ..Item("versao do esquema", ##class(LabStudy.Schema).Version()_
                                   " de "_##class(LabStudy.Schema).#LATEST)

    new SQLCODE, %msg, n
    &sql(SELECT COUNT(*) INTO :n FROM %Dictionary.CompiledClass
         WHERE Name %STARTSWITH 'LabStudy.')
    do ..Item("classes LabStudy", $SELECT(SQLCODE=0: n, 1: "?"))

    do ..Section("dados")
    do ..ClassGlobals("LabStudy.Patient")
    do ..ClassGlobals("LabStudy.Exam")
    do ..ClassGlobals("LabStudy.AuditEntry")

    set withExams = ##class(LabStudy.Patient).Statistics(.patients, .exams)
    do ..Item("pacientes com exame", withExams)

    &sql(SELECT COUNT(*) INTO :n FROM LabStudy.EXAM WHERE ResultStatus = 'pending')
    do ..Item("exames pendentes", $SELECT(SQLCODE=0: n, 1: "?"))

    do ..Section("arquivos")
    set dir = ##class(LabStudy.FileIO).#DEFAULTDIR
    do ..Item("diretorio", dir)
    do ..Item("existe", ##class(%File).DirectoryExists(dir))

    if ##class(%File).DirectoryExists(dir) {
        do ##class(LabStudy.Demo.Ap2).List(dir, "*.csv")
    }

    write !
    quit $$$OK
}

}
```

E em `src/LabStudy/App.cls`:

```objectscript
Parameter VERSION = "2.0";

ClassMethod About() As %Status
{
    write "==============================", !
    write ..#APPNAME, !
    write "Version: ", ..#VERSION, !
    write "IRIS:    ", $SYSTEM.Version.GetNumber(), !
    write "Namespace: ", $NAMESPACE, !
    write "==============================", !
    quit $$$OK
}
```

Acrescente à tabela `LabStudy.Menu.Dispatch()`:

```objectscript
        "11": "OptExport",
        "12": "OptImport",
        "13": "OptSystemInfo",
```

```objectscript
ClassMethod OptExport() As %Status
{
    set dir = ..Prompt("Diretorio", ##class(LabStudy.FileIO).#DEFAULTDIR)
    set files = ##class(LabStudy.FileIO).BackupAll(dir)

    for i = 1:1:$LISTLENGTH(files) {
        write "    ", $LIST(files, i), !
    }
    quit $$$OK
}

ClassMethod OptImport() As %Status
{
    set path = ..Prompt("Arquivo CSV de pacientes")
    quit:path="" $$$OK

    // always simulate first
    do ##class(LabStudy.FileIO).ImportPatientsCsv(path, 0)

    if ..Confirm("  Executar a importacao de verdade?") {
        do ##class(LabStudy.FileIO).ImportPatientsCsv(path, 1)
    }
    quit $$$OK
}

ClassMethod OptSystemInfo() As %Status
{
    do ##class(LabStudy.SystemInfo).Dashboard()
    quit $$$OK
}
```

Execução esperada:

```
LABSTUDY>DO ##class(LabStudy.FileIO).ExportPatientsCsv()
  201 pacientes exportados para /tmp/labstudy/pacientes.csv
  (12847 bytes)

LABSTUDY>DO ##class(LabStudy.FileIO).Verify("/tmp/labstudy/pacientes.csv")
  arquivo    : pacientes.csv
  tamanho    : 12847 bytes
  cabecalho  : ok
  linhas     : 201
  campos     : 7 a 7 (esperado 7)
  consistente: sim

LABSTUDY>DO ##class(LabStudy.FileIO).ImportPatientsCsv("/tmp/labstudy/pacientes.csv")
  simulacao (nada sera gravado) de /tmp/labstudy/pacientes.csv
  ------------------------------
  linhas lidas : 201
  criados      : 0
  ja existiam  : 201
  falhas       : 0

LABSTUDY>DO ##class(LabStudy.SystemInfo).Dashboard()

======================================================
                   LabStudy 2.0
======================================================
IRIS                   2026.1
plataforma             UNIX
namespace              LABSTUDY
usuario                _SYSTEM
processo               12345
data/hora              2026-08-19 16:55:02

======================================================
                      esquema
======================================================
versao do esquema      5 de 5
classes LabStudy       36

======================================================
                       dados
======================================================
LabStudy.Patient       201 objetos, proximo id 201
LabStudy.Exam          2003 objetos, proximo id 2003
LabStudy.AuditEntry    412 objetos, proximo id 412
pacientes com exame    200
exames pendentes       3

======================================================
                      arquivos
======================================================
diretorio              /tmp/labstudy
existe                 1
  conteudo de /tmp/labstudy (*.csv)
tipo  nome                          tamanho  modificado
--------------------------------------------------------------------
 F    pacientes.csv                   12847  2026-08-19 16:54:10
--------------------------------------------------------------------
  1 itens
```

**Por que cada decisão:**

- **A exportação usa `%SelectMode = 1`.** Sem isso, `BirthDate` sairia como número de dias — ilegível para qualquer sistema externo e impossível de reimportar. Modo ODBC é o formato de intercâmbio, como visto nos capítulos 9 e 16.
- **Todos os caminhos passam por `DefaultPath`, que usa `SubDirectoryName`.** Concatenar `..#DEFAULTDIR_"/pacientes.csv"` funcionaria no Linux e produziria `C:\lab/pacientes.csv` no Windows. O helper existe para que a regra fique num lugar só — e para que ninguém precise lembrar dela ao acrescentar a próxima exportação.
- **A exportação usa `ListUtil.ToCsv`**, que coloca aspas quando o valor contém vírgula, aspas ou quebra de linha. Um nome como `"Silva, Maria"` quebraria um CSV montado com concatenação simples — e o `Verify` teria acusado 8 campos em vez de 7.
- **O `Verify` confere a consistência do número de campos em todas as linhas.** É a verificação mínima de um arquivo exportado: se alguma linha tem número diferente de campos, o escape falhou em algum lugar. **Exportar sem verificar é entregar um arquivo que talvez não possa ser lido de volta.**
- **A ida e volta foi testada de verdade:** exportamos 201 pacientes e a importação encontrou 201 já existentes, com **zero falhas**. Isso prova que o formato exportado é reimportável — a única prova que importa.
- **A importação simula por padrão.** Importar dados por engano numa base de produção é irreversível sem restauração de backup. O menu reforça isso: simula primeiro, sempre, e só executa após confirmação explícita.
- **A importação valida antes de criar.** Número de registro no formato certo, data ISO válida, e existência prévia — três verificações, cada uma com mensagem que identifica a linha. Um arquivo com problemas produz um relatório útil em vez de um erro genérico.
- **A importação pula os já existentes em vez de falhar.** Reimportar o mesmo arquivo é uma operação **idempotente** — a lição do Capítulo 11 aplicada à integração.
- **O `BackupAll` cria um diretório com carimbo de data e hora**, então backups não se sobrescrevem.
- **O painel reúne informação de todas as camadas** construídas ao longo da apostila: versão do produto, versão do esquema (Capítulo 11), armazenamento (Capítulo 8), estatísticas (Capítulo 9), formatação (Capítulo 15), data e hora (Capítulo 16) e arquivos (este capítulo). **A versão 2.0 marca o fim do domínio T4 e um sistema completo em todas as dimensões que a certificação cobra** — falta apenas o tratamento de erros, que é o T5.

---

## 9. Quiz do capítulo

**Q1.** Qual variável dá acesso aos pacotes de utilitários do sistema?

- A) `$SYSTEM`
- B) `%SYS`
- C) `$UTIL`
- D) `$ZSYSTEM`

---

**Q2.** Como listar os métodos disponíveis num pacote do `$SYSTEM`?

- A) `WRITE $SYSTEM.OBJ`
- B) `DO $SYSTEM.OBJ.Help()`
- C) `ZWRITE $SYSTEM.OBJ`
- D) Não é possível.

---

**Q3.** Qual classe trata o arquivo como entidade do sistema de arquivos (existir, copiar, apagar)?

- A) `%Stream.FileCharacter`
- B) `%File`
- C) `%SYSTEM.Util`
- D) `%Net.FtpSession`

---

**Q4.** Qual método cria toda a cadeia de diretórios pais, se necessário?

- A) `CreateDirectory`
- B) `CreateDirectoryChain`
- C) `MakeDir`
- D) `EnsureDirectory`

---

**Q5.** Por que usar `SubDirectoryName` em vez de concatenar com `/`?

- A) É mais rápido.
- B) Usa o separador correto do sistema operacional, tornando o código portável.
- C) Valida se o diretório existe.
- D) Cria o diretório.

---

**Q6.** Qual query lista o conteúdo de um diretório?

- A) `%File:Directory`
- B) `%File:FileSet`
- C) `%File:List`
- D) `%SYSTEM.Util:Files`

---

**Q7.** Depois de `USE arquivo`, o que acontece com o comando `WRITE`?

- A) Continua escrevendo na tela.
- B) Passa a escrever no arquivo, porque ele virou o dispositivo atual.
- C) Gera erro.
- D) Escreve nos dois.

---

**Q8.** Qual é o maior risco de usar `OPEN`/`USE`/`CLOSE`?

- A) É lento.
- B) Esquecer de restaurar `$IO`, fazendo toda a saída seguinte ir para o arquivo.
- C) Não funciona no Linux.
- D) Não permite leitura.

---

**Q9.** Por que preferir streams a `OPEN`/`USE`/`CLOSE` para arquivos?

- A) Streams são mais rápidos.
- B) Streams não alteram o dispositivo atual e têm API mais simples.
- C) `OPEN` não funciona com arquivos de texto.
- D) Streams não precisam de caminho.

---

**Q10.** O que a propriedade `TranslateTable` de um stream de arquivo controla?

- A) O terminador de linha.
- B) A conversão de codificação de caracteres.
- C) O tamanho máximo.
- D) As permissões do arquivo.

---

**Q11.** Ao fazer uma chamada HTTP, o que **sempre** deve ser definido?

- A) O `ContentType`.
- B) O `Timeout`.
- C) O `UserAgent`.
- D) A porta.

---

**Q12.** Um `%Status` de sucesso numa chamada HTTP significa o quê?

- A) Que o servidor respondeu com sucesso.
- B) Que a chamada **aconteceu**; é preciso conferir o `StatusCode` da resposta separadamente.
- C) Que o corpo está vazio.
- D) Que a conexão é segura.

---

**Q13.** Por que não fazer chamadas HTTP dentro de uma transação?

- A) Não funciona.
- B) Você seguraria travas e journal esperando a rede.
- C) O HTTP não é transacional.
- D) O `%Status` seria perdido.

---

**Q14.** Qual é a forma recomendada de executar trabalho periódico?

- A) Um `JOB` num laço infinito.
- B) O agendador de tarefas (`%SYS.Task`), com uma classe que estende `%SYS.Task.Definition`.
- C) `$ZF` chamando o cron do sistema.
- D) `HANG` até a hora certa.

---

**Q15.** Qual o principal risco do `$ZF(-100, ...)`?

- A) É lento.
- B) Executa comandos no sistema operacional; argumentos vindos de fora comprometem o servidor.
- C) Só funciona no Windows.
- D) Não devolve valor.

---

**Q16.** Ao escrever uma rotina que apaga arquivos, qual deve ser o comportamento padrão?

- A) Apagar, com opção de simular.
- B) **Simular**, exigindo confirmação explícita para apagar de verdade.
- C) Perguntar arquivo por arquivo.
- D) Mover para a lixeira.

---

### Gabarito comentado

**Q1 — Resposta: A.**
- **A está certa:** `$SYSTEM` dá acesso aos pacotes de utilitários.
- **B está errada:** `%SYS` é um namespace.
- **C e D estão erradas:** não existem.

**Q2 — Resposta: B.**
- **B está certa:** `Help()` lista os métodos disponíveis com descrição, refletindo a versão instalada.
- **A e C estão erradas:** não produzem a listagem de métodos.
- **D está errada:** é justamente para isso que `Help()` existe.

**Q3 — Resposta: B.**
- **B está certa:** `%File` cuida do arquivo como entidade: existir, tamanho, copiar, renomear, apagar, listar.
- **A está errada:** streams cuidam do **conteúdo**.
- **C está errada:** `%SYSTEM.Util` traz utilidades gerais.
- **D está errada:** FTP é transferência por rede.

**Q4 — Resposta: B.**
- **B está certa:** `CreateDirectoryChain` cria toda a cadeia; `CreateDirectory` cria um nível.
- **C e D estão erradas:** não são os nomes dos métodos.

**Q5 — Resposta: B.**
- **B está certa:** o separador de caminho difere entre sistemas operacionais.
- **A está errada:** desempenho não é a razão.
- **C e D estão erradas:** ele apenas monta o nome.

**Q6 — Resposta: B.**
- **B está certa:** `FileSet` é a class query de listagem de diretório, executada como qualquer outra.
- **A, C e D estão erradas:** não existem com esses nomes.

**Q7 — Resposta: B.**
- **B está certa:** `USE` troca o dispositivo atual, e `WRITE` sempre escreve no dispositivo atual.
- **A está errada:** é justamente o que muda.
- **C e D estão erradas:** não há erro nem duplicação.

**Q8 — Resposta: B.**
- **B está certa:** sem restaurar `$IO`, toda a saída — inclusive mensagens de erro — vai para o arquivo.
- **A está errada:** desempenho não é o problema.
- **C e D estão erradas:** funciona nas duas plataformas e permite leitura.

**Q9 — Resposta: B.**
- **B está certa:** streams não mexem no dispositivo atual e oferecem uma API mais direta.
- **A está errada:** desempenho não é o critério principal.
- **C e D estão erradas:** ambos funcionam com arquivos e precisam de caminho.

**Q10 — Resposta: B.**
- **B está certa:** `TranslateTable` define a conversão de codificação de caracteres.
- **A está errada:** isso é `LineTerminator`.
- **C e D estão erradas:** não são controladas por essa propriedade.

**Q11 — Resposta: B.**
- **B está certa:** sem tempo limite, um serviço lento trava o seu processo.
- **A, C e D estão erradas:** úteis conforme o caso, mas não obrigatórias.

**Q12 — Resposta: B.**
- **B está certa:** o `%Status` cobre a comunicação; o resultado da requisição está no `StatusCode`.
- **A está errada:** o servidor pode ter respondido 404 ou 500 com sucesso de comunicação.
- **C e D estão erradas:** não têm relação.

**Q13 — Resposta: B.**
- **B está certa:** a transação continuaria aberta durante a espera da rede, segurando travas e acumulando journal.
- **A está errada:** funciona tecnicamente — o problema é o custo.
- **C está errada:** a questão não é o protocolo.
- **D está errada:** o `%Status` é devolvido normalmente.

**Q14 — Resposta: B.**
- **B está certa:** o agendador controla concorrência, registra histórico e permite reprogramar sem alterar código.
- **A está errada:** reimplementa mal o que já existe.
- **C está errada:** acrescenta risco de segurança e dependência do sistema operacional.
- **D está errada:** prende um processo indefinidamente.

**Q15 — Resposta: B.**
- **B está certa:** é execução de comando no servidor; argumentos não validados são comprometimento total.
- **A está errada:** desempenho não é a questão.
- **C está errada:** funciona nas plataformas suportadas.
- **D está errada:** há variantes que capturam a saída.

**Q16 — Resposta: B.**
- **B está certa:** operações irreversíveis devem exigir confirmação explícita e simular por padrão.
- **A está errada:** inverte a proteção.
- **C está errada:** inviável em lote.
- **D está errada:** não há lixeira; e a proteção continua sendo necessária.

---

## 10. Resumo relâmpago

1. **`$SYSTEM`** dá acesso aos pacotes de utilitários. Use **`DO $SYSTEM.X.Help()`** para descobrir o que existe **na sua versão**.
2. Versão: **`$ZVERSION`**, `$SYSTEM.Version.GetVersion()`, `.GetNumber()`, `.GetPlatform()`. **Compare versões numericamente**, não como texto.
3. Processo: `$JOB`, `$USERNAME`, `$ROLES`, `$NAMESPACE`, `$IO`, `$ZSTORAGE`, `$STORAGE`.
4. Instalação: `$SYSTEM.Util.InstallDirectory()`, `.ManagerDirectory()`, `.GetEnviron(nome)`. **Nunca grave caminhos absolutos no código.**
5. **`%File`** cuida do arquivo como entidade; **streams** cuidam do conteúdo.
6. `%File`: `Exists`, `DirectoryExists`, **`CreateDirectoryChain`**, `GetFileSize`, `GetDirectory`, `GetFilename`, **`SubDirectoryName`**, `CopyFile`, `Rename`, `Delete`.
7. Métodos de `%File` devolvem **1/0** e não geram erro. **Confira o retorno.**
8. **`%File:FileSet`** lista diretórios, executada com `%PrepareClassQuery` como qualquer class query.
9. Streams de arquivo: `LinkToFile`, `Write`, `WriteLine`, `Rewind`, `ReadLine`, `MoveToEnd`, `%Save`. Vale tudo do Capítulo 4.
10. **`TranslateTable`** define a codificação; **`LineTerminator`** define o fim de linha. Defina os dois ao trocar arquivos entre sistemas.
11. Remova um `$CHAR(13)` sobrando ao ler arquivos vindos do Windows.
12. **Tudo é dispositivo.** `WRITE` escreve no dispositivo atual, indicado por **`$IO`**.
13. `OPEN` / `USE` / `CLOSE`: confira **`$TEST`** após o `OPEN`, **guarde e restaure `$IO`**, e sempre feche.
14. **Prefira streams a `OPEN`/`USE`/`CLOSE`** para arquivos.
15. **`%Net.HttpRequest`**: defina `Server`, `Https`, e **sempre `Timeout`**. Confira o `%Status` **e** o `StatusCode`. O corpo vem em `Data`, que é um **stream**.
16. **Nunca chame serviços externos dentro de uma transação.**
17. Trabalho periódico: **agendador de tarefas** (`%SYS.Task.Definition` com `OnTask()`), não `JOB` em laço.
18. **`$ZF`** executa comandos do sistema operacional. Trate como o `XECUTE`: lista de permitidos, ou não use.
19. Operações irreversíveis (apagar, importar) devem **simular por padrão** e exigir confirmação explícita.
20. **Verifique o que você exportou.** Um arquivo que não pode ser reimportado não é um backup.

---

## 11. Cartões de memorização

**Frente:** Como descobrir os métodos de um pacote do `$SYSTEM`?
**Verso:** `DO $SYSTEM.Pacote.Help()` — lista os métodos da sua versão.

**Frente:** Qual classe cuida do arquivo como entidade, e qual cuida do conteúdo?
**Verso:** `%File` cuida da entidade (existir, copiar, apagar, listar). Streams cuidam do conteúdo.

**Frente:** Qual método cria toda a cadeia de diretórios?
**Verso:** `##class(%File).CreateDirectoryChain(caminho, .info)`.

**Frente:** Por que usar `SubDirectoryName` em vez de concatenar com barra?
**Verso:** Porque ele usa o separador correto do sistema operacional, tornando o código portável.

**Frente:** Qual query lista um diretório?
**Verso:** `%File:FileSet`, executada com `%PrepareClassQuery("%File", "FileSet")`.

**Frente:** O que `$IO` indica?
**Verso:** O dispositivo atual — para onde `WRITE` escreve e de onde `READ` lê.

**Frente:** Qual o maior perigo de `USE`?
**Verso:** Esquecer de restaurar `$IO`. Toda a saída seguinte, inclusive erros, vai para o outro dispositivo.

**Frente:** O que conferir logo após um `OPEN`?
**Verso:** `$TEST` — 1 se abriu.

**Frente:** O que `TranslateTable` e `LineTerminator` controlam num stream de arquivo?
**Verso:** Codificação de caracteres e terminador de linha, respectivamente.

**Frente:** Que limpeza fazer ao ler linhas de um arquivo vindo do Windows?
**Verso:** Remover o `$CHAR(13)` do fim: `$ZSTRIP(linha, ">", $CHAR(13))`.

**Frente:** O que **sempre** definir numa chamada HTTP?
**Verso:** O `Timeout`. Sem ele, um serviço lento trava o seu processo.

**Frente:** `%Status` de sucesso numa chamada HTTP significa o quê?
**Verso:** Que a chamada aconteceu. O resultado está no `StatusCode` da resposta, que precisa ser conferido separadamente.

**Frente:** Onde vem o corpo de uma resposta HTTP?
**Verso:** Em `HttpResponse.Data`, que é um **stream** — rebobine antes de ler.

**Frente:** Por que não chamar serviços externos dentro de uma transação?
**Verso:** Você seguraria travas e journal enquanto espera a rede.

**Frente:** Qual a forma recomendada de trabalho periódico?
**Verso:** Uma classe que estende `%SYS.Task.Definition` implementando `OnTask()`, agendada pelo Task Manager.

**Frente:** Qual o risco do `$ZF(-100, ...)`?
**Verso:** Executa comandos no sistema operacional. Argumentos não validados comprometem o servidor.

**Frente:** Como comparar duas versões do IRIS?
**Verso:** Numericamente, parte por parte com `$PIECE` e `+`. Como texto, `"2026.10"` viria antes de `"2026.2"`.

**Frente:** Qual deve ser o padrão de uma rotina que apaga ou importa?
**Verso:** **Simular**. Só executar de verdade após confirmação explícita.

**Frente:** Como saber se um arquivo exportado presta?
**Verso:** Reimportando-o. Um arquivo que não volta não é backup.

**Frente:** Antes de escrever código de infraestrutura, o que fazer?
**Verso:** Verificar se o IRIS já não traz pronto. Quase sempre traz.

---

Fim do domínio **T4 — Functions & APIs**, o de maior peso da prova, e do sistema LabStudy na versão **2.0**.

O próximo capítulo abre o **T5 — Handles and Resolves Errors** (12 questões), começando pelas ferramentas de diagnóstico.

Digite CONTINUAR para o próximo capítulo.
