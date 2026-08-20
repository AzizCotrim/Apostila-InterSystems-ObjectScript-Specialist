# Apostila InterSystems ObjectScript Specialist
## Capítulo 0 — Preparando o ambiente

---

## 1. Objetivo do capítulo

Ao terminar este capítulo, você será capaz de:

1. Explicar, com suas palavras, o que é o **InterSystems IRIS**, o que é um **namespace**, o que é o **Terminal** e o que é uma **classe**.
2. Instalar o **IRIS Community Edition** no seu computador (por container ou por instalador nativo) e confirmar que ele está no ar.
3. Abrir o **Portal de Gerenciamento** (Management Portal) e o **Terminal**.
4. Criar o seu **namespace de estudos**, chamado `LABSTUDY`.
5. Instalar e conectar o **VS Code** ao seu IRIS.
6. Escrever, **salvar**, **compilar** e **executar** o seu primeiro código ObjectScript.
7. Reconhecer e ler em voz alta os símbolos estranhos da linguagem: `$`, `^`, `#`, `##class`, `%`, `..`, `_`, `!`.
8. Seguir a **receita de execução** que vamos usar em todos os exercícios da apostila, do primeiro ao último capítulo.
9. Iniciar o **projeto contínuo** da apostila: um pequeno sistema de laboratório.

Nada aqui exige conhecimento prévio. Se você nunca programou, comece por aqui mesmo e vá devagar.

---

## 1.1. Como esta apostila se relaciona com a prova

A certificação organiza o conteúdo em **cinco domínios**, e cada domínio tem tópicos numerados no formato `T1.1`, `T4.3`, e assim por diante. Esta apostila cobre **um tópico por capítulo**, na mesma ordem.

Quando um capítulo remete a outro, ele usa **o número do capítulo** — sempre de 0 a 22, nunca a notação da prova. Guarde este mapa para consultar o edital sem se perder:

| Domínio | Peso | Tópico | Capítulo |
|---|---|---|---|
| **T1 — Objects** | 23 questões | T1.1 Uses Classes | **1** |
| | | T1.2 Creates Properties and Indexes | **2** |
| | | T1.3 Creates ObjectScript Methods | **3** |
| | | T1.4 Uses Complex Structures | **4** |
| **T2 — Data Integrity** | 15 questões | T2.1 Ensures Data Integrity | **5** |
| | | T2.2 Tracks Application Data | **6** |
| | | T2.3 Implements Security Features | **7** |
| **T3 — IRIS Features** | 14 questões | T3.1 Differentiates Storage Media | **8** |
| | | T3.2 Leverages ObjectScript/SQL Features | **9** |
| | | T3.3 Handles Nulls | **10** |
| | | T3.4 Handles Schema Evolution | **11** |
| | | T3.5 Ensures Scalability and Performance | **12** |
| **T4 — Functions & APIs** | **26 questões** | T4.1 Traverses and Sorts Arrays | **13** |
| | | T4.2 Manipulates Lists | **14** |
| | | T4.3 Manipulates Strings | **15** |
| | | T4.4 Mathematical, Logical, Date/Time | **16** |
| | | T4.5 Decision and Control Structures | **17** |
| | | T4.6 Executes Methods and Queries Objects | **18** |
| | | T4.7 Uses APIs for Common Operations | **19** |
| **T5 — Errors** | 12 questões | T5.1 Uses Troubleshooting Tools | **20** |
| | | T5.2 Handles and Logs Runtime Errors | **21** |
| | | T5.3 Recognizes and Resolves Common Errors | **22** |

Repare no peso: **T4 sozinho vale mais de um terço da prova**. Os capítulos 13 a 19 merecem, proporcionalmente, mais tempo de estudo que os demais.

---

## 2. O conceito em linguagem de gente

### 2.1 O que é o InterSystems IRIS

Imagine que você vai montar um laboratório de análises clínicas. Você precisaria de:

- um **arquivo** gigante para guardar fichas de pacientes e resultados (onde os dados ficam);
- um **jeito de escrever as regras** do laboratório: quem pode ver o quê, como se calcula um resultado, o que acontece quando chega uma amostra;
- um **balcão** onde as pessoas e outros sistemas conversam com o seu laboratório.

Normalmente você compraria três coisas separadas para isso. O **IRIS** é o laboratório inteiro montado numa coisa só: ele é ao mesmo tempo o lugar onde os dados moram, o lugar onde o seu código mora e roda, e o lugar que atende as conexões de fora.

Guarde esta frase: **o IRIS não é um programa que você abre e usa. É um serviço que fica ligado, esperando você mandar coisas para ele.** Você escreve código, entrega esse código ao IRIS, e o IRIS guarda e executa.

Um IRIS instalado e ligado no seu computador se chama uma **instância**. O nome padrão da instância costuma ser `IRIS`. Você pode ter mais de uma instância na mesma máquina, com nomes diferentes, sem uma atrapalhar a outra.

### 2.2 O que é um namespace

Pense num prédio com várias salas. Cada sala tem uma placa na porta com um nome. Dentro de cada sala há dois armários:

- um armário de **dados** (as fichas, os resultados, os cadastros);
- um armário de **código** (as regras, os programas).

Uma **sala** dessas é um **namespace**. O nome do namespace é a placa na porta.

Quando você está trabalhando, você está **dentro de uma sala por vez**. Se você criar um paciente estando na sala `LABSTUDY`, esse paciente fica no armário da sala `LABSTUDY`. Se depois você entrar na sala `USER` e procurar o paciente, você não vai encontrar — ele está na outra sala.

Isso explica o erro mais comum de quem começa: *"eu criei a classe e ela sumiu"*. Ela não sumiu. Você mudou de sala.

Duas salas já vêm prontas na instalação e você precisa conhecê-las:

- **`USER`** — a sala de rascunho, livre para você usar.
- **`%SYS`** — a sala da administração do prédio. É onde ficam as configurações do IRIS, os usuários, as senhas. **Não é lugar de estudo.** Você vai entrar lá pouquíssimas vezes, e sempre com cuidado.

E existe uma diferença importante que muita gente confunde:

- **Database** (base de dados) é o **armário físico**: um arquivo em disco chamado `IRIS.DAT`, dentro de uma pasta.
- **Namespace** é a **sala**: um nome que aponta para um ou mais armários.

Ou seja: o namespace é um apelido organizado; o database é onde os bytes realmente ficam. Um namespace normalmente aponta para um armário de dados e um armário de código (que podem até ser o mesmo armário). Vários namespaces podem apontar para o mesmo database — isso se chama **mapeamento**, e vamos ver com calma no Capítulo 3.

### 2.3 O que é o Terminal

O **Terminal** é o balcão de atendimento do IRIS. É uma janela preta onde você digita uma ordem, aperta Enter, e o IRIS obedece na hora.

Não tem botão, não tem menu. Só uma linha esperando você. Essa linha se chama **prompt**, e ela mostra em que sala você está:

```
USER>
```

Isso quer dizer: *"você está na sala USER, pode mandar"*.

O Terminal é a ferramenta mais importante do seu estudo. É onde você testa uma ideia em três segundos, vê o resultado e entende o que aconteceu. Na prova de certificação, boa parte das questões mostra exatamente isso: um comando e o que apareceria na tela.

### 2.4 O que é uma classe

Pense na **forma de bolo**. A forma não é o bolo. A forma é o molde: ela define o formato, o tamanho, quantos furos tem. Com uma forma você faz vários bolos iguais no formato, mas diferentes no recheio.

- A **classe** é a forma.
- O **objeto** é o bolo que saiu daquela forma.

Uma classe chamada `Patient` diria: *"todo paciente tem um nome, uma data de nascimento e um número de registro"*. Cada paciente de verdade — João, Maria, Ana — é um **objeto** feito a partir dessa classe.

Dentro de uma classe existem três tipos de conteúdo que você vai usar o tempo todo:

- **Propriedades** (*properties*): as informações que o objeto guarda. Nome, idade, resultado.
- **Métodos** (*methods*): as ações que o objeto sabe fazer. Calcular a idade, imprimir a ficha, validar um resultado.
- **Parâmetros** (*parameters*): valores fixos, decididos na hora de compilar, que valem para a classe inteira. Por exemplo, a versão do sistema.

E existe uma classe que é **mãe** de quase tudo no IRIS: `%RegisteredObject`. Quando você escreve `Extends %RegisteredObject`, você está dizendo: *"minha classe herda tudo que essa classe base já sabe fazer"*. É como comprar uma forma de bolo que já vem com antiaderente de fábrica: você não precisou inventar o antiaderente.

No Capítulo 1 vamos abrir cada uma dessas ideias em detalhe. Por enquanto, basta saber que **classe é o molde e objeto é a peça feita no molde**.

### 2.5 Onde o seu código mora

Você não guarda o seu código só em arquivos soltos no computador. Você **entrega** o código ao IRIS, e o IRIS guarda dentro do namespace.

Existem três tipos de "coisa de código" que você vai encontrar:

| Tipo | Extensão | O que é |
|---|---|---|
| Classe | `.cls` | O molde, com propriedades e métodos. É o formato principal e moderno. |
| Rotina | `.mac` | Um arquivo de código mais antigo e mais direto, sem classe em volta. Ainda muito usado. |
| Include | `.inc` | Um arquivo só com definições reaproveitáveis (macros). |

E existe uma palavra que você vai ouvir o tempo todo: **compilar**.

Compilar é o IRIS pegar o texto que você escreveu e transformar em algo que ele consegue executar. **Enquanto você não compila, o IRIS não conhece o seu código.** Salvar o arquivo não basta. Esse é, disparado, o erro número um de quem está começando.

Analogia: escrever a receita num papel é salvar. Levar o papel para a cozinha e o cozinheiro decorar a receita é compilar. Se você só escreveu o papel e deixou na gaveta, e depois pediu o prato, a cozinha responde: *"não conheço esse prato"*.

---

## 3. A sintaxe explicada

Antes de instalar qualquer coisa, vamos decifrar os símbolos. Assim, quando eles aparecerem, você não vai travar.

### 3.1 A forma geral de um comando

```
COMANDO argumento1,argumento2
```

Regras, uma por vez:

**a) O nome do comando não diferencia maiúsculas de minúsculas.**
`WRITE`, `write` e `Write` são a mesma coisa. Nesta apostila, o código de exemplo usa minúsculas dentro de métodos e MAIÚSCULAS quando estamos digitando direto no Terminal — é só um estilo para você diferenciar rapidamente o que é para digitar no balcão.

**b) Entre o comando e o argumento vai EXATAMENTE UM espaço.**

Isto funciona:

```
WRITE "Hello"
```

Isto **dá erro**:

```
WRITE  "Hello"
```

Por quê? Porque no ObjectScript, **dois espaços depois do comando significam "este comando não tem argumento"**. Então o IRIS entende: *"WRITE sem nada, e depois um novo comando chamado `"Hello"`"*. Como não existe comando com esse nome, ele reclama com `<SYNTAX>`.

Guarde: **um espaço separa; dois espaços encerram.**

**c) Vários comandos podem ficar na mesma linha, separados por espaço.**

```
SET x = 10 WRITE x
```

Isso faz duas coisas: guarda 10 em `x` e escreve `x` na tela.

**d) Nomes de variáveis, classes, métodos e propriedades DIFERENCIAM maiúsculas de minúsculas.**

`name` e `Name` são **duas variáveis diferentes**. Isso cai na prova. Cuidado.

**e) A vírgula separa argumentos.**

```
WRITE "Hello, ", name, "!"
```

Isso escreve as três coisas em sequência, sem espaço extra entre elas.

### 3.2 Os símbolos, um por um

Agora sim: cada símbolo estranho, explicado literalmente.

---

**`$` — o cifrão**

O cifrão marca **coisas que já vêm prontas dentro do IRIS**. Você não escreveu, ele já existe.

Ele aparece de duas formas:

1. **Função do sistema**: tem parênteses e recebe valores.
   `$LENGTH("abc")` devolve 3.
   Leia como: *"cifrão LENGTH de abc"*.

2. **Variável especial do sistema**: não tem parênteses, é um valor que o IRIS mantém sozinho.
   `$NAMESPACE` devolve o nome da sala em que você está.
   `$HOROLOG` devolve a data e hora atuais no formato interno do IRIS.

Detalhe importante: **não pode haver espaço entre o `$` e o nome**. `$ LENGTH` está errado.

Existe também `$$$` (três cifrões), que marca uma **macro** — um apelido que é trocado por outro texto na hora de compilar. Você vai usar `$$$OK` já neste capítulo. E existe `$$` (dois cifrões), que chama uma função escrita dentro de uma rotina. O `$$$` aparece a partir do Capítulo 1; o `$$`, no Capítulo 18, junto com rotinas.

---

**`^` — o acento circunflexo**

O circunflexo marca uma **global**.

Uma **global** é uma variável que **fica gravada em disco**. Ela não some quando você fecha o Terminal, não some quando você desliga o computador. Ela é o armário de verdade.

```
SET ^Config("version") = "1.0"
```

Isso gravou permanentemente. Amanhã, em outra sessão, `^Config("version")` continua valendo `"1.0"`.

Compare com:

```
SET version = "1.0"
```

Sem o circunflexo, é uma **variável local**: mora na memória, existe só durante a sua sessão e desaparece quando você sai.

Regra mental: **circunflexo = disco. Sem circunflexo = memória.**

O circunflexo também aparece num segundo uso, para indicar uma rotina: `DO ^MyRoutine` significa *"execute a rotina chamada MyRoutine"*. O contexto (se está depois de `DO` ou dentro de uma expressão) diz qual dos dois usos é.

---

**`%` — o sinal de porcentagem**

O `%` marca **itens que pertencem ao sistema**, não a você.

- `%String`, `%Integer`, `%Status` são tipos de dado que o IRIS já traz prontos.
- `%RegisteredObject`, `%Persistent` são classes base do IRIS.
- `%SYS` é o namespace de administração.

Regra prática de sobrevivência: **nunca comece os seus próprios nomes com `%`.** Esse espaço é reservado. Se você criar uma variável chamada `%x`, ela terá um comportamento especial de escopo que só vamos estudar mais adiante, e por enquanto isso só traz confusão.

Quando você vir `%` na frente de um nome, leia: *"isso é do IRIS, não é meu"*.

---

**`#` — a cerquilha**

O `#` marca **instruções para o compilador**, não para a execução. São ordens dadas *antes* do código rodar.

- `#Include %occInclude` — *"antes de compilar, traga as definições deste arquivo"*.
- `#Define MAXROWS 100` — *"crie a macro MAXROWS valendo 100"*.
- `#Dim patient As LabStudy.Patient` — *"compilador, saiba que esta variável é deste tipo"*. Isso não muda o que acontece na execução; serve para o compilador e para o editor te ajudarem com autocompletar e avisos.

E há um uso diferente, com dois pontos antes: `..#VERSION` significa *"o parâmetro VERSION da minha própria classe"*. Vamos usar isso no projeto contínuo.

---

**`##class(...)` — dois jogos da velha seguidos de class**

Esta é a forma de **apontar para uma classe pelo nome completo**.

```
##class(LabStudy.Hello).SayHello()
```

Leia em voz alta: *"da classe LabStudy.Hello, execute o método SayHello"*.

Quebrando em pedaços:

- `##class(` — abertura fixa, sempre escrita assim;
- `LabStudy.Hello` — o nome completo da classe, com o **pacote** antes do ponto;
- `)` — fecha;
- `.SayHello()` — o ponto seguido do nome do método e dos parênteses com os argumentos.

O que é **pacote** (*package*)? É a parte do nome antes do último ponto. Serve para organizar, como pastas. Em `LabStudy.Model.Patient`, o pacote é `LabStudy.Model` e a classe é `Patient`. O IRIS não cria pastas de verdade; o ponto é apenas parte do nome. Mas ele usa isso para agrupar as classes na tela.

---

**`..` — dois pontos finais seguidos**

Dois pontos significam **"meu próprio"**, ou seja, algo da própria classe ou do próprio objeto onde o código está.

- `..Name` — a propriedade `Name` deste objeto.
- `..Validate()` — o método `Validate` desta mesma classe.
- `..#VERSION` — o parâmetro `VERSION` desta mesma classe.

Você só pode usar `..` **dentro** de um método da classe. Fora dela, use `##class(...)`.

---

**`_` — o sublinhado**

O sublinhado **cola dois textos**. Chama-se concatenação.

```
SET fullName = "Ana" _ " " _ "Silva"
```

`fullName` agora vale `"Ana Silva"`.

Não confunda com `+`. O `+` soma números. `"10" + "5"` dá `15`. `"10" _ "5"` dá `"105"`.

---

**`!` — a exclamação dentro do WRITE**

Dentro de um `WRITE`, a exclamação significa **"pule para a próxima linha"**.

```
WRITE "linha um", !, "linha dois"
```

Sai assim:

```
linha um
linha dois
```

E `?10` dentro de um `WRITE` significa *"vá para a coluna 10"*. Serve para alinhar colunas na tela.

---

**Comentários**

Um comentário é um texto para humanos que o IRIS ignora. Existem quatro formas:

```
// comentário de uma linha (a forma mais usada)
; comentário de uma linha (forma antiga, também válida)
#; comentário que some antes mesmo da compilação
/* comentário
   de várias
   linhas */
```

Atenção à mesma regra do espaço: um comentário `//` que vem depois de um comando na mesma linha precisa estar separado por espaço.

---

### 3.3 A forma geral de uma classe

```
Class Pacote.Nome Extends ClasseMae
{

ClassMethod NomeDoMetodo(param As Tipo) As TipoDeRetorno
{
    // corpo do método
}

}
```

Explicando parte por parte:

- `Class` — palavra fixa. Anuncia que ali começa a definição de uma classe. **Obrigatória.**
- `Pacote.Nome` — o nome completo. **Obrigatório.** Precisa ser exatamente igual ao nome do arquivo (`Pacote/Nome.cls`).
- `Extends ClasseMae` — de quem essa classe herda. **Opcional**, mas na prática quase sempre presente.
- `{ ... }` — as chaves delimitam o corpo da classe. **Obrigatórias.**
- `ClassMethod` — anuncia um método que pertence à **classe**, não a um objeto específico. Chama-se pelo nome da classe, sem precisar criar objeto.
- `(param As Tipo)` — a lista de parâmetros que o método recebe. **Opcional** — pode ser `()` vazio.
- `As TipoDeRetorno` — o tipo do valor devolvido. **Opcional**; se o método não devolve nada, omita.
- As chaves do método delimitam o corpo. **Obrigatórias.**

Existe também `Method` (sem "Class" na frente), que é um método que só funciona **em cima de um objeto já criado**. A diferença entre `ClassMethod` e `Method` é o assunto do Capítulo 3 e cai bastante na prova.

---

## 4. Exemplo comentado

Vamos ao seu primeiro código. Leia agora, execute na seção 7.

### 4.1 No Terminal

```
USER>WRITE "Hello, IRIS!", !
Hello, IRIS!
USER>SET name = "Aziz"
USER>WRITE "Hello, ", name, "!", !
Hello, Aziz!
USER>WRITE $NAMESPACE, !
USER
USER>WRITE $LENGTH(name), !
4
USER>ZWRITE name
name="Aziz"
USER>HALT
```

Linha por linha:

- `WRITE "Hello, IRIS!", !` — o comando `WRITE` escreve na tela. O texto entre aspas duplas é uma **string** (um pedaço de texto). A vírgula separa os argumentos. O `!` quebra a linha ao final.
- `SET name = "Aziz"` — o comando `SET` guarda um valor numa variável. À esquerda do `=` fica o nome; à direita, o valor. Não aparece nada na tela, porque `SET` não escreve, só guarda.
- `WRITE "Hello, ", name, "!", !` — três argumentos: um texto fixo, o conteúdo da variável (sem aspas, senão sairia a palavra "name") e outro texto fixo. Depois o `!` para pular linha.
- `WRITE $NAMESPACE, !` — `$NAMESPACE` é a variável especial que guarda o nome da sala atual.
- `WRITE $LENGTH(name), !` — `$LENGTH` é uma função do sistema que devolve o tamanho de um texto. "Aziz" tem 4 caracteres.
- `ZWRITE name` — `ZWRITE` é o irmão investigativo do `WRITE`: em vez de só mostrar o valor, ele mostra **o nome e o valor formatados**. É a sua ferramenta favorita para depurar. Guarde este comando; você vai usá-lo centenas de vezes.
- `HALT` — encerra a sessão do Terminal e fecha a janela.

### 4.2 A primeira classe

Este é o arquivo `LabStudy/Hello.cls`:

```objectscript
/// This is the first class of the study material.
/// Triple-slash comments become the class documentation.
Class LabStudy.Hello Extends %RegisteredObject
{

/// Prints a greeting to the current device.
ClassMethod SayHello(name As %String = "World") As %Status
{
    write "Hello, ", name, "!", !
    write "Current namespace: ", $NAMESPACE, !
    quit $$$OK
}

}
```

Linha por linha:

- `/// This is the first class...` — três barras é um **comentário de documentação**. O IRIS guarda esse texto e mostra na documentação automática da classe. Duas barras (`//`) seriam um comentário comum, que o IRIS ignora completamente.
- `Class LabStudy.Hello Extends %RegisteredObject` — declara a classe. Pacote `LabStudy`, nome `Hello`. Herda de `%RegisteredObject`, a classe base que dá à sua classe a capacidade de existir como objeto na memória.
- `{` — abre o corpo da classe.
- `ClassMethod SayHello(...)` — método de classe. Pode ser chamado direto pelo nome da classe.
- `name As %String = "World"` — um parâmetro chamado `name`, do tipo `%String` (texto), com **valor padrão** `"World"`. Se quem chamar não passar nada, `name` vale `"World"`.
- `As %Status` — o método devolve um `%Status`. `%Status` é o tipo padrão do IRIS para dizer *"deu certo"* ou *"deu errado, e o motivo foi este"*. Vamos estudar `%Status` a fundo no Capítulo 5.
- `write "Hello, ", name, "!", !` — escreve a saudação.
- `write "Current namespace: ", $NAMESPACE, !` — mostra em que sala o código está rodando. Útil para provar que você está no lugar certo.
- `quit $$$OK` — o comando `quit` **encerra o método e devolve um valor**. `$$$OK` é uma macro que representa "sucesso". Ela já está disponível dentro de classes sem você precisar incluir nada.
- `}` fecha o método, `}` fecha a classe.

### 4.3 Como executar essa classe

Depois de **salvar e compilar**, no Terminal:

```
LABSTUDY>DO ##class(LabStudy.Hello).SayHello()
Hello, World!
Current namespace: LABSTUDY

LABSTUDY>DO ##class(LabStudy.Hello).SayHello("Aziz")
Hello, Aziz!
Current namespace: LABSTUDY
```

O comando `DO` executa alguma coisa e **descarta o valor devolvido**. Como aqui só queremos ver o texto na tela, `DO` está de bom tamanho.

Se você quisesse ver o valor devolvido, usaria `WRITE` no lugar de `DO`:

```
LABSTUDY>WRITE ##class(LabStudy.Hello).SayHello("Aziz")
Hello, Aziz!
Current namespace: LABSTUDY
1
```

Aquele `1` no final é o valor de `$$$OK`. Sucesso, no IRIS, é representado pelo número 1.

---

## 5. Variações e detalhes

### 5.1 Instalação — Caminho A: container (recomendado)

Este é o caminho mais limpo: nada é instalado de verdade no seu sistema, e para começar do zero basta apagar o container.

**Pré-requisito:** ter o Docker Desktop instalado e rodando.

**Passo 1 — baixar e subir a imagem:**

```bash
docker run --name iris-study -d --init \
  -p 1972:1972 \
  -p 52773:52773 \
  containers.intersystems.com/intersystems/iris-community:latest-em
```

Explicando cada pedaço:

- `docker run` — cria e inicia um container.
- `--name iris-study` — o apelido do container. Você vai usar esse nome nos próximos comandos.
- `-d` — roda em segundo plano (*detached*), sem prender o seu terminal.
- `--init` — recomendado pela InterSystems para que o container encerre corretamente.
- `-p 1972:1972` — abre a **porta do superserver**, por onde ferramentas externas (inclusive o VS Code, em alguns modos) conversam com o IRIS.
- `-p 52773:52773` — abre a **porta web**, por onde você acessa o Portal de Gerenciamento pelo navegador.
- a última linha é o endereço da imagem oficial Community Edition.

> **Sobre a etiqueta da imagem:** as etiquetas de versão (`latest-em`, `latest-cd`, ou uma versão específica como `2025.1`) mudam com o tempo. Se o comando acima falhar dizendo que a imagem não existe, **verificar na documentação oficial** da InterSystems a etiqueta atual. O restante do comando não muda.

**Passo 2 — conferir se subiu:**

```bash
docker ps
```

Você deve ver o container `iris-study` com status `Up` e, depois de um minuto, `(healthy)`.

**Passo 3 — abrir o Terminal dentro do container:**

```bash
docker exec -it iris-study iris session IRIS
```

- `docker exec -it iris-study` — *"execute algo dentro do container iris-study, em modo interativo"*.
- `iris session IRIS` — *"abra uma sessão de Terminal na instância chamada IRIS"*.

Na primeira vez ele pede usuário e senha. Use `_SYSTEM` e a senha `SYS`. O IRIS vai **obrigar você a trocar a senha** imediatamente. Escolha uma senha e **anote**, porque você vai precisar dela no Portal e no VS Code.

Para sair do Terminal: `HALT` e Enter.

Para desligar e religar o IRIS depois:

```bash
docker stop iris-study
docker start iris-study
```

### 5.2 Instalação — Caminho B: instalador nativo no Windows

1. Acesse o site de avaliação da InterSystems (`evaluation.intersystems.com`), crie uma conta gratuita e baixe o kit do **IRIS Community Edition** para Windows.
2. Execute o instalador. Aceite a instalação **Development** quando ele perguntar o tipo de instalação.
3. Deixe o nome da instância como `IRIS` e as portas nos valores padrão.
4. Ao final, aparece um **ícone azul do IRIS na bandeja do sistema**, ao lado do relógio.
5. Clique nesse ícone. O menu que abre tem, entre outras opções:
   - **Terminal** — abre o balcão de comandos;
   - **Management Portal** — abre o Portal no navegador;
   - **Start IRIS / Stop IRIS** — liga e desliga o serviço.

Se o menu mostrar que a instância está parada, clique em **Start IRIS** e espere.

### 5.3 O Portal de Gerenciamento

O Portal é a interface web de administração. Abra no navegador:

```
http://localhost:52773/csp/sys/UtilHome.csp
```

Entre com `_SYSTEM` e a sua senha.

O que você precisa saber navegar por enquanto:

- **System Explorer → Classes** — ver as classes de um namespace.
- **System Explorer → SQL** — rodar consultas SQL.
- **System Explorer → Globals** — abrir os armários e ver os dados crus.
- **System Administration → Configuration → System Configuration → Namespaces** — criar e ver namespaces.

No canto superior você sempre vê **em qual namespace o Portal está**, com um link para trocar. Confira sempre. É outra fonte clássica de confusão.

### 5.4 Criando o namespace `LABSTUDY`

Você poderia estudar dentro do `USER`, mas separar é melhor: se algo der muito errado, você apaga a sala inteira e recomeça sem perder mais nada.

No Portal:

1. **System Administration → Configuration → System Configuration → Namespaces**.
2. Clique em **Create New Namespace**.
3. Em **Name of the namespace**, digite `LABSTUDY`. Por convenção, nomes de namespace são em MAIÚSCULAS.
4. Em **Select an existing database for Globals**, clique em **Create New Database**.
5. Dê o nome `LABSTUDYDATA`. O Portal sugere uma pasta automaticamente; aceite a sugestão. Avance até **Finish**.
6. De volta à tela do namespace, em **Select an existing database for Routines**, escolha o mesmo `LABSTUDYDATA`. Para estudo, dados e código no mesmo armário é o mais simples.
7. Deixe marcada a opção de criar o namespace com as configurações padrão e clique em **Save**.

Confirme no Terminal:

```
USER>ZN "LABSTUDY"
LABSTUDY>WRITE $NAMESPACE, !
LABSTUDY
```

O comando `ZN` troca de namespace. Repare que **o nome vai entre aspas duplas** — `ZN LABSTUDY` sem aspas dá erro.

### 5.5 Instalando e conectando o VS Code

1. Instale o VS Code.
2. Na aba de extensões (o ícone de blocos na barra lateral), procure por **InterSystems ObjectScript Extension Pack** e instale. Esse pacote traz três extensões de uma vez: a de ObjectScript, o Language Server (autocompletar e destaque de sintaxe) e o Server Manager (gerenciador de conexões).
3. Crie uma pasta no seu computador para o estudo, por exemplo `C:\iris-study` (ou `~/iris-study`).
4. Dentro dela crie uma subpasta chamada `src`.
5. Abra a pasta `C:\iris-study` no VS Code (**File → Open Folder**).
6. Crie o arquivo `.vscode/settings.json` dentro dela com este conteúdo:

```json
{
  "intersystems.servers": {
    "iris-study": {
      "webServer": {
        "scheme": "http",
        "host": "localhost",
        "port": 52773
      },
      "username": "_SYSTEM"
    }
  },
  "objectscript.conn": {
    "active": true,
    "server": "iris-study",
    "ns": "LABSTUDY"
  },
  "objectscript.export": {
    "folder": "src",
    "addCategory": false
  }
}
```

Explicando:

- `intersystems.servers` — a lista de servidores IRIS conhecidos. Aqui definimos um chamado `iris-study`, que responde em `localhost` na porta web `52773` com o usuário `_SYSTEM`.
- `objectscript.conn` — qual servidor usar nesta pasta, e **em qual namespace**. Repare no `"ns": "LABSTUDY"`: se você esquecer isso, seu código vai parar em outro namespace.
- `objectscript.export.folder` — a subpasta onde o código exportado do servidor é gravado.

7. Salve. O VS Code vai pedir a senha do `_SYSTEM` na primeira conexão. Digite a senha que você definiu.
8. Na barra de status, embaixo, deve aparecer o nome do servidor e o namespace. Se aparecer riscado ou com aviso, a conexão falhou — confira porta, senha e se o IRIS está ligado.

**Como o VS Code compila:** por padrão, a extensão compila **ao salvar** o arquivo. Você salva com `Ctrl+S` e a compilação acontece sozinha. Se der erro de compilação, a mensagem aparece no painel **Output**, escolhendo o canal **ObjectScript**. Você também pode compilar manualmente pela paleta de comandos (`Ctrl+Shift+P`) digitando `ObjectScript: Compile`.

**Onde salvar os arquivos:** o nome do arquivo tem que espelhar o nome da classe. A classe `LabStudy.Hello` fica em:

```
C:\iris-study\src\LabStudy\Hello.cls
```

Ou seja: cada ponto do pacote vira uma pasta, e o último pedaço vira o nome do arquivo com extensão `.cls`.

### 5.6 O Terminal dentro do VS Code

Além do Terminal do sistema, versões recentes da extensão permitem abrir um terminal do IRIS direto no VS Code, pela paleta de comandos (`Ctrl+Shift+P` → procure por *Terminal* nos comandos que começam com **ObjectScript** ou **InterSystems**). A disponibilidade e o nome exato desse comando variam conforme a versão da extensão e a versão do servidor; se não aparecer na sua instalação, **verificar na documentação oficial** e continuar usando o Terminal do sistema ou o `docker exec`. Nada nesta apostila depende desse recurso.

### 5.7 Abreviações de comandos

Todo comando do ObjectScript tem uma forma abreviada. Você vai encontrar isso em código antigo e **na prova**:

| Completo | Abreviado | O que faz |
|---|---|---|
| `WRITE` | `W` | escreve na tela |
| `SET` | `S` | atribui valor |
| `DO` | `D` | executa |
| `QUIT` | `Q` | encerra o bloco/método |
| `KILL` | `K` | apaga variável |
| `NEW` | `N` | cria escopo novo para a variável |
| `IF` | `I` | condição |
| `FOR` | `F` | repetição |
| `ZWRITE` | `ZW` | mostra nome e valor |

Nesta apostila vamos sempre escrever por extenso, porque é mais legível. Mas **saiba ler as abreviações**, porque questões de prova as usam.

### 5.8 Limites do Community Edition

O IRIS Community Edition é gratuito e completo em recursos de linguagem, mas tem **limites de licença** (número de núcleos usados, número de conexões simultâneas e volume de dados) e **não pode ser usado em produção**. Para estudar e fazer todos os exercícios desta apostila, ele é mais do que suficiente. Os números exatos dos limites mudam entre versões: **verificar na documentação oficial** se precisar do dado preciso.

---

## 6. Pegadinhas e erros comuns

**1) Você salvou mas não compilou.**
Sintoma: ao rodar, aparece `<CLASS DOES NOT EXIST>`.
Causa: o IRIS ainda não conhece a classe.
Solução: compile. No VS Code, salve com a conexão ativa; confira o painel Output.

**2) Você está no namespace errado.**
Sintoma: `<CLASS DOES NOT EXIST>` mesmo depois de compilar; ou a classe aparece no Portal mas some quando você troca de sala.
Causa: compilou em `USER` e está rodando em `LABSTUDY`, ou o contrário.
Solução: `WRITE $NAMESPACE` no Terminal e confira o `"ns"` do `settings.json`.

**3) Dois espaços depois do comando.**
Sintoma: `<SYNTAX>`.
Causa: `WRITE  "oi"` — com dois espaços, o IRIS lê "WRITE sem argumento" e depois tenta executar `"oi"` como se fosse um comando.
Solução: exatamente um espaço.

**4) Maiúsculas e minúsculas em nomes.**
Sintoma: `<UNDEFINED>` numa variável que você jurou ter criado.
Causa: você fez `SET name = "Aziz"` e depois `WRITE Name`. São duas variáveis diferentes.
Solução: padronize. Escolha um estilo e mantenha.

**5) `ZN` sem aspas.**
Sintoma: erro de sintaxe.
Causa: `ZN LABSTUDY`.
Solução: `ZN "LABSTUDY"`. O nome do namespace é uma string.

**6) Nome da classe diferente do caminho do arquivo.**
Sintoma: a compilação falha ou cria uma classe com nome inesperado.
Causa: arquivo em `src/Lab/Hello.cls` mas a classe declarada como `LabStudy.Hello`.
Solução: `src/LabStudy/Hello.cls` para a classe `LabStudy.Hello`. Pacote = pasta.

**7) Esquecer o `!` no `WRITE`.**
Sintoma: tudo sai grudado numa linha só.
Causa: `WRITE` não pula linha sozinho.
Solução: termine com `, !`.

**8) Usar `HALT` achando que é `QUIT`, ou o contrário.**
`HALT` encerra a sessão inteira do Terminal e fecha a janela. `QUIT` encerra apenas o bloco de código atual (um método, um laço). No prompt do Terminal, quem fecha a janela é `HALT`.

**9) Confundir `_` com `+`.**
`"10" + "5"` dá `15` (soma). `"10" _ "5"` dá `"105"` (cola). A prova adora essa diferença.

**10) Achar que o IRIS "abre" como um aplicativo.**
Sintoma: você fecha o Terminal e acha que desligou o IRIS.
Realidade: o IRIS é um serviço que continua ligado. Fechar o Terminal só fecha o balcão, não o laboratório.

**11) Porta 52773 ocupada.**
Sintoma: o container não sobe ou o Portal não abre.
Causa: outra coisa já usa essa porta.
Solução: mapeie outra porta no `docker run`, por exemplo `-p 52774:52773`, e use `localhost:52774` no navegador e no `settings.json`.

**12) Perder a senha do `_SYSTEM`.**
Sintoma: você não consegue mais entrar no Portal nem conectar o VS Code.
Solução mais rápida no caminho de container: apagar o container (`docker rm -f iris-study`) e criar de novo. Você perde o que criou dentro dele — por isso, nesta fase, **anote a senha**.

---

## 7. MÃO NA MASSA

> **Receita de execução — vale para TODOS os exercícios desta apostila.**
>
> 1. Verifique que o IRIS está ligado (`docker ps`, ou o ícone azul na bandeja).
> 2. Abra o Terminal (`docker exec -it iris-study iris session IRIS`, ou pelo menu do ícone).
> 3. Entre na sala de estudos: `ZN "LABSTUDY"`.
> 4. Se o exercício pede uma classe: escreva o arquivo no VS Code, em `src/`, salve (`Ctrl+S`) e confira que compilou sem erro.
> 5. Volte ao Terminal e execute com `DO ##class(Pacote.Classe).Metodo()`.
> 6. Compare o que apareceu na tela com o que o exercício diz que deveria aparecer.

---

### Exercício 0.1 — Primeiro contato com o Terminal

**a) Enunciado:** Abra o Terminal, descubra em qual namespace você está, guarde o seu nome numa variável, imprima uma saudação usando essa variável e depois inspecione a variável com `ZWRITE`. Por fim, encerre a sessão.

**b) Dica:** Você precisa de quatro comandos: `WRITE`, `SET`, `ZWRITE` e `HALT`. A variável especial do namespace começa com cifrão.

**c) Como testar:** Digite cada linha e aperte Enter. Você deve ver o nome do namespace, a saudação com o seu nome e a linha do `ZWRITE` mostrando `nome="valor"`. Ao final, a janela do Terminal deve fechar ou voltar para o prompt do sistema.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

```
USER>WRITE $NAMESPACE, !
USER

USER>SET studentName = "Aziz"

USER>WRITE "Welcome, ", studentName, "!", !
Welcome, Aziz!

USER>ZWRITE studentName
studentName="Aziz"

USER>HALT
```

**Por que cada decisão:**

- `WRITE $NAMESPACE, !` — sempre comece confirmando onde você está. Isso vira reflexo e economiza horas de confusão. O `!` no final deixa a saída limpa.
- `SET studentName = "Aziz"` — o nome da variável está em inglês e sem acento, seguindo a regra da apostila (código em inglês). Nada aparece na tela porque `SET` só guarda.
- `WRITE "Welcome, ", studentName, "!", !` — o texto fixo vai entre aspas; a variável vai **sem** aspas. Se você escrevesse `"studentName"`, sairia a palavra literal.
- `ZWRITE studentName` — a diferença para o `WRITE` é que o `ZWRITE` mostra também o **nome** da variável. Quando um dia você tiver dez variáveis e não souber qual está errada, `ZWRITE` sem argumento nenhum mostra todas de uma vez. Experimente.
- `HALT` — encerra a sessão.

---

### Exercício 0.2 — Criando e usando o namespace de estudos

**a) Enunciado:** Crie o namespace `LABSTUDY` pelo Portal de Gerenciamento, com um database novo chamado `LABSTUDYDATA`. Depois, no Terminal, entre nesse namespace e prove que você está lá. Grave uma global de teste e confirme que ela existe.

**b) Dica:** O caminho no Portal é System Administration → Configuration → System Configuration → Namespaces. No Terminal, o comando de troca de sala é `ZN` e o nome vai entre aspas. Global é a variável com circunflexo na frente.

**c) Como testar:** Depois de `ZN "LABSTUDY"`, o prompt deve mudar de `USER>` para `LABSTUDY>`. A global deve aparecer no `ZWRITE`. Bônus: abra o Portal em System Explorer → Globals, troque para o namespace `LABSTUDY` e veja a global `^StudyLog` listada ali.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

Pelo Portal, siga exatamente os passos da seção 5.4.

No Terminal:

```
USER>ZN "LABSTUDY"

LABSTUDY>WRITE $NAMESPACE, !
LABSTUDY

LABSTUDY>SET ^StudyLog("chapter0") = "environment ready"

LABSTUDY>ZWRITE ^StudyLog
^StudyLog("chapter0")="environment ready"

LABSTUDY>HALT
```

Agora feche tudo, abra o Terminal de novo, entre em `LABSTUDY` e rode `ZWRITE ^StudyLog` outra vez. **O valor continua lá.** Isso é a diferença prática entre global e variável local: se você tivesse feito `SET studyLog = "..."` sem o circunflexo, o valor teria desaparecido ao fechar a sessão.

**Por que cada decisão:**

- Namespace separado protege o `USER` e deixa você apagar tudo sem medo depois.
- Usar o mesmo database para dados e código é a configuração mais simples para estudo. Em ambientes reais, é comum separar — e o Capítulo 3 explica por quê.
- `^StudyLog("chapter0")` mostra que uma global pode ter **subscritos** (o texto entre parênteses). Pense num armário com gavetas etiquetadas. Vamos explorar isso a fundo no Capítulo 13.

---

### Exercício 0.3 — Sua primeira classe compilada

**a) Enunciado:** Crie a classe `LabStudy.Hello` no VS Code, com um método de classe `SayHello` que recebe um nome (com valor padrão `"World"`), imprime uma saudação e o namespace atual, e devolve sucesso. Compile e execute das duas formas: sem argumento e com o seu nome.

**b) Dica:** O arquivo tem que ficar em `src/LabStudy/Hello.cls`. Valor padrão de parâmetro se escreve com `= "World"` logo depois do tipo. Sucesso é `$$$OK`.

**c) Como testar:** Salve com `Ctrl+S` e confira o painel Output (canal ObjectScript) — deve aparecer uma mensagem de compilação bem-sucedida. No Terminal, dentro de `LABSTUDY`, rode as duas chamadas. Você deve ver `Hello, World!` na primeira e `Hello, Aziz!` na segunda, e nas duas o namespace `LABSTUDY`.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

Arquivo `src/LabStudy/Hello.cls`:

```objectscript
/// First class of the study material.
Class LabStudy.Hello Extends %RegisteredObject
{

/// Prints a greeting and the current namespace.
ClassMethod SayHello(name As %String = "World") As %Status
{
    write "Hello, ", name, "!", !
    write "Current namespace: ", $NAMESPACE, !
    quit $$$OK
}

}
```

No Terminal:

```
LABSTUDY>DO ##class(LabStudy.Hello).SayHello()
Hello, World!
Current namespace: LABSTUDY

LABSTUDY>DO ##class(LabStudy.Hello).SayHello("Aziz")
Hello, Aziz!
Current namespace: LABSTUDY
```

**Por que cada decisão:**

- `Extends %RegisteredObject` — mesmo que aqui a gente só use `ClassMethod`, herdar dessa classe base é o hábito correto para classes de aplicação que não gravam dados em disco. Uma classe que **grava** herdaria de `%Persistent`, e isso é o Capítulo 1.
- `ClassMethod` em vez de `Method` — porque não faz sentido criar um objeto só para dar bom dia. Método de classe se chama direto pelo nome da classe.
- `name As %String = "World"` — valor padrão evita erro quando ninguém passa argumento. Repare que o padrão fica **depois** do tipo.
- `As %Status` e `quit $$$OK` — mesmo sem precisar do retorno agora, este é o padrão da InterSystems: métodos que "fazem alguma coisa" devolvem um `%Status`. Adotar isso desde o primeiro dia deixa você alinhado com o que a prova espera.
- `$NAMESPACE` dentro do método — serve de prova visual de que o código está executando na sala certa.

---

### Exercício 0.4 — Provando que compilar é obrigatório

**a) Enunciado:** Faça um experimento de propósito. Adicione um segundo método à classe `LabStudy.Hello`, chamado `SayGoodbye`, que imprima uma despedida. **Não salve ainda.** Vá ao Terminal e tente executá-lo. Observe o erro. Depois salve (compile) e execute de novo.

**b) Dica:** O erro que você deve ver começa com `<METHOD DOES NOT EXIST>`.

**c) Como testar:** Antes de salvar, o Terminal recusa. Depois de salvar, funciona. O objetivo é você **sentir** a diferença entre escrever e compilar, e nunca mais esquecer.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

O método a acrescentar, dentro das chaves da classe:

```objectscript
/// Prints a goodbye message.
ClassMethod SayGoodbye(name As %String = "World") As %Status
{
    write "Goodbye, ", name, "!", !
    quit $$$OK
}
```

Antes de salvar:

```
LABSTUDY>DO ##class(LabStudy.Hello).SayGoodbye()

DO ##class(LabStudy.Hello).SayGoodbye()
^
<METHOD DOES NOT EXIST> *SayGoodbye,LabStudy.Hello
```

Depois de salvar com `Ctrl+S`:

```
LABSTUDY>DO ##class(LabStudy.Hello).SayGoodbye()
Goodbye, World!
```

**Por que cada decisão:**

- O erro `<METHOD DOES NOT EXIST>` traz duas informações preciosas: o nome do método procurado e a classe em que ele foi procurado. Aprenda a ler isso; economiza muito tempo.
- Repare que a classe **existia** (senão o erro seria `<CLASS DOES NOT EXIST>`). O que faltava era o método. Distinguir esses dois erros é assunto do Capítulo 22 e cai na prova.

---

### Exercício 0.5 — PROJETO CONTÍNUO: a fundação do LabStudy

Este é o primeiro tijolo do projeto que vamos construir capítulo a capítulo.

**O projeto:** um pequeno sistema de laboratório de análises clínicas, escrito em ObjectScript puro. Ao longo da apostila ele vai ganhar pacientes, exames, resultados, validações, transações, tratamento de erros e consultas. Ao final, você terá um sistema pequeno mas completo — e cada linha dele terá sido escrita por você.

**a) Enunciado:** Crie a classe `LabStudy.App`, que será o "cartão de visita" do sistema. Ela deve ter:

1. Um **parâmetro** chamado `VERSION` com o valor `"0.1"`.
2. Um **parâmetro** chamado `APPNAME` com o valor `"LabStudy Laboratory System"`.
3. Um `ClassMethod` chamado `About` que imprime, em linhas separadas, o nome do sistema, a versão e o namespace atual, e devolve sucesso.
4. Um `ClassMethod` chamado `CheckEnvironment` que grava na global `^LabStudyLog` a marca de que o ambiente foi verificado, e depois imprime o conteúdo dessa global.

**b) Dica:** Parâmetro se declara com a palavra `Parameter` e se lê, dentro da própria classe, com `..#NOME`. Para gravar na global, use `SET ^LabStudyLog(...) = ...`. Para mostrar a global, `ZWRITE ^LabStudyLog` — mas dentro de um método você escreve `zwrite` em minúsculas normalmente, como qualquer comando.

**c) Como testar:**

```
LABSTUDY>DO ##class(LabStudy.App).About()
LABSTUDY>DO ##class(LabStudy.App).CheckEnvironment()
```

O primeiro deve mostrar as três linhas de identificação. O segundo deve mostrar a linha da global gravada. Feche o Terminal, abra de novo e rode `ZWRITE ^LabStudyLog`: o registro tem que continuar lá.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

Arquivo `src/LabStudy/App.cls`:

```objectscript
/// Entry point and identity of the LabStudy laboratory system.
/// This class grows along the study material.
Class LabStudy.App Extends %RegisteredObject
{

/// Current version of the system.
Parameter VERSION = "0.1";

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

}
```

Saída esperada:

```
LABSTUDY>DO ##class(LabStudy.App).About()
==============================
LabStudy Laboratory System
Version: 0.1
Namespace: LABSTUDY
==============================

LABSTUDY>DO ##class(LabStudy.App).CheckEnvironment()
Environment log:
^LabStudyLog("environment","checked")=1
^LabStudyLog("environment","version")="0.1"
```

**Por que cada decisão:**

- **`Parameter` em vez de variável.** Um `Parameter` é decidido na **compilação** e vale para a classe inteira. A versão do sistema não muda enquanto o programa roda — então ela é um parâmetro, não uma variável. Colocar a versão espalhada em textos soltos pelo código seria pior: quando ela mudasse, você teria que caçar todos os lugares.
- **`..#VERSION`.** Os dois pontos dizem "da minha própria classe" e a cerquilha diz "isto é um parâmetro". Junto: *"o parâmetro VERSION da minha própria classe"*. Se você escrevesse `..VERSION` (sem cerquilha), o IRIS procuraria uma **propriedade** com esse nome e não acharia.
- **Ponto e vírgula depois do `Parameter`.** Diferente dos métodos, a declaração de parâmetro termina com `;`. Esquecer isso é erro de compilação.
- **Global com dois subscritos.** `^LabStudyLog("environment", "checked")` agrupa a informação por assunto. Quando o log crescer, tudo que for de ambiente estará debaixo do mesmo ramo. Isso é o começo da modelagem em globais, que aprofundamos no Capítulo 8.
- **`zwrite` de uma global inteira.** Sem subscrito, o `zwrite` percorre tudo que está debaixo daquela global e imprime cada nó. É a forma mais rápida de inspecionar dados no IRIS.
- **Duas responsabilidades, dois métodos.** `About` identifica; `CheckEnvironment` verifica e registra. Um método, um propósito. O projeto vai crescer muito, e essa disciplina desde o primeiro arquivo evita bagunça depois.

---

## 8. Quiz do capítulo

**Q1.** Você digitou no Terminal e recebeu um erro:

```
USER>WRITE  "Hello"
```

Qual é a causa?

- A) Faltou o `!` no final.
- B) Há dois espaços entre o comando e o argumento.
- C) `WRITE` precisa de parênteses.
- D) Strings devem usar aspas simples.

---

**Q2.** Qual é a diferença entre `SET total = 10` e `SET ^total = 10`?

- A) Nenhuma; o circunflexo é apenas estilo.
- B) O primeiro grava em disco e o segundo em memória.
- C) O primeiro fica na memória da sessão e o segundo é gravado em disco.
- D) O segundo só funciona no namespace `%SYS`.

---

**Q3.** Um desenvolvedor compilou a classe `LabStudy.Hello` no namespace `USER` e, em seguida, no Terminal executou:

```
LABSTUDY>DO ##class(LabStudy.Hello).SayHello()
```

O que acontece?

- A) Funciona normalmente; classes são globais a toda a instância.
- B) Erro `<CLASS DOES NOT EXIST>`, porque a classe está em outro namespace.
- C) Erro `<SYNTAX>`, porque falta o `WRITE`.
- D) A classe é copiada automaticamente para `LABSTUDY`.

---

**Q4.** O que `##class(LabStudy.App)` representa?

- A) A criação de um novo objeto da classe.
- B) Uma referência à classe `LabStudy.App`, para chamar seus membros.
- C) Um comentário de documentação.
- D) A importação do pacote `LabStudy`.

---

**Q5.** Considere:

```
SET a = "20"
SET b = "3"
WRITE a + b, "-", a _ b
```

O que aparece na tela?

- A) `23-203`
- B) `203-23`
- C) `23-23`
- D) Erro, porque não se soma texto.

---

**Q6.** Dentro de um método da classe `LabStudy.App`, que tem `Parameter VERSION = "0.1";`, qual expressão lê corretamente esse parâmetro?

- A) `..VERSION`
- B) `$VERSION`
- C) `..#VERSION`
- D) `^VERSION`

---

**Q7.** Qual afirmação sobre maiúsculas e minúsculas está correta?

- A) Tudo diferencia maiúsculas de minúsculas.
- B) Nada diferencia maiúsculas de minúsculas.
- C) Nomes de comandos não diferenciam, mas nomes de variáveis diferenciam.
- D) Nomes de variáveis não diferenciam, mas nomes de comandos diferenciam.

---

**Q8.** Você quer trocar do namespace atual para `LABSTUDY` no Terminal. Qual comando está correto?

- A) `ZN LABSTUDY`
- B) `SET $NAMESPACE = LABSTUDY`
- C) `ZN "LABSTUDY"`
- D) `USE "LABSTUDY"`

---

**Q9.** Qual é a diferença entre `WRITE` e `ZWRITE` ao inspecionar uma variável?

- A) `ZWRITE` mostra o nome e o valor formatados; `WRITE` mostra só o valor.
- B) `ZWRITE` grava em disco; `WRITE` não.
- C) `ZWRITE` só funciona com globais.
- D) Não há diferença.

---

**Q10.** Uma classe declara `ClassMethod Run() As %Status`. Você executa `DO ##class(Pkg.C).Run()` e nada aparece na tela, mas você sabe que o método devolve `$$$OK`. Por quê?

- A) Porque `$$$OK` não devolve valor.
- B) Porque `DO` descarta o valor devolvido pelo método.
- C) Porque o método não foi compilado.
- D) Porque `%Status` não pode ser exibido.

---

**Q11.** Onde deve ficar o arquivo da classe `LabStudy.Model.Patient`, considerando `objectscript.export.folder` igual a `src`?

- A) `src/LabStudy.Model.Patient.cls`
- B) `src/LabStudy/Model/Patient.cls`
- C) `src/Patient/LabStudy/Model.cls`
- D) Qualquer lugar; o caminho não importa.

---

**Q12.** Você adicionou um método novo no VS Code, mas ao executar recebe `<METHOD DOES NOT EXIST>`. Qual é a causa mais provável?

- A) O namespace foi apagado.
- B) O arquivo não foi salvo/compilado.
- C) O IRIS está desligado.
- D) O método precisa ser `Method` em vez de `ClassMethod`.

---

### Gabarito comentado

**Q1 — Resposta: B.**
- **B está certa:** dois espaços depois do comando fazem o IRIS interpretar `WRITE` como comando sem argumento e tratar `"Hello"` como um novo comando, o que gera `<SYNTAX>`.
- **A está errada:** o `!` é opcional; sem ele a saída só não pula linha.
- **C está errada:** `WRITE` não usa parênteses para seus argumentos.
- **D está errada:** no ObjectScript, strings usam **aspas duplas**; aspas simples não delimitam string.

**Q2 — Resposta: C.**
- **C está certa:** sem circunflexo é variável local (memória, some ao fim da sessão); com circunflexo é global (disco, permanece).
- **A está errada:** a diferença é fundamental, não estética.
- **B está errada:** inverte os papéis.
- **D está errada:** globais funcionam em qualquer namespace.

**Q3 — Resposta: B.**
- **B está certa:** código compilado vive dentro de um namespace. Em `LABSTUDY` aquela classe não existe.
- **A está errada:** classes não são compartilhadas automaticamente entre namespaces (isso só ocorre com mapeamento explícito, assunto do Capítulo 3).
- **C está errada:** a sintaxe do comando está correta; `DO` executa e descarta o retorno.
- **D está errada:** o IRIS não copia código sozinho.

**Q4 — Resposta: B.**
- **B está certa:** `##class(Nome)` é a sintaxe de referência a uma classe, usada para chamar métodos de classe e outros membros.
- **A está errada:** criar objeto seria `##class(Nome).%New()`, que veremos no Capítulo 1.
- **C está errada:** comentário de documentação é `///`.
- **D está errada:** o ObjectScript não usa `##class` para importar pacotes.

**Q5 — Resposta: A.**
- **A está certa:** `a + b` converte os textos em números e soma, dando `23`. Depois sai o traço. Depois `a _ b` cola os textos, dando `203`. Resultado: `23-203`.
- **B está errada:** inverte a ordem das duas operações.
- **C está errada:** trata `_` como se fosse soma.
- **D está errada:** o ObjectScript converte texto numérico automaticamente na aritmética.

**Q6 — Resposta: C.**
- **C está certa:** `..` significa "da minha própria classe" e `#` indica que o membro é um parâmetro.
- **A está errada:** `..VERSION` procuraria uma **propriedade**, não um parâmetro.
- **B está errada:** o cifrão é para funções e variáveis especiais do sistema.
- **D está errada:** o circunflexo indica global, não parâmetro de classe.

**Q7 — Resposta: C.**
- **C está certa:** `WRITE` e `write` são equivalentes; `name` e `Name` são variáveis distintas.
- **A está errada:** comandos e funções do sistema não diferenciam.
- **B está errada:** nomes de variáveis, classes, métodos e propriedades diferenciam.
- **D está errada:** inverte a regra.

**Q8 — Resposta: C.**
- **C está certa:** `ZN` troca de namespace e recebe o nome como **string**, entre aspas duplas.
- **A está errada:** sem aspas o IRIS não interpreta como string e ocorre erro.
- **B está errada:** ainda que exista uma forma de atribuir `$NAMESPACE`, a expressão apresentada está sem aspas e não é a forma direta esperada no Terminal.
- **D está errada:** `USE` serve para selecionar dispositivo de entrada e saída, não namespace.

**Q9 — Resposta: A.**
- **A está certa:** `ZWRITE` imprime nome e valor num formato próprio para inspeção; `WRITE` imprime apenas o valor.
- **B está errada:** nenhum dos dois grava dados; quem grava é `SET` em global.
- **C está errada:** `ZWRITE` funciona com variáveis locais e globais.
- **D está errada:** a diferença de formato é justamente o motivo de `ZWRITE` existir.

**Q10 — Resposta: B.**
- **B está certa:** `DO` executa e ignora o valor devolvido. Para ver o retorno, usa-se `WRITE`.
- **A está errada:** `$$$OK` representa o valor de sucesso, que é `1`.
- **C está errada:** se não estivesse compilado, apareceria erro de método ou classe inexistente.
- **D está errada:** `%Status` é um valor comum e pode ser exibido.

**Q11 — Resposta: B.**
- **B está certa:** cada ponto do pacote vira uma pasta e a última parte vira o arquivo `.cls`.
- **A está errada:** o nome com pontos num arquivo só não corresponde à estrutura esperada de pastas.
- **C está errada:** a ordem dos elementos está trocada.
- **D está errada:** o caminho precisa espelhar o nome da classe.

**Q12 — Resposta: B.**
- **B está certa:** enquanto não compila, o servidor não conhece o método novo. A classe existe (por isso o erro é de método, não de classe), mas a versão compilada é a antiga.
- **A está errada:** namespace apagado geraria falha de conexão ou erro diferente.
- **C está errada:** com o IRIS desligado você nem teria prompt para executar o comando.
- **D está errada:** o tipo do método não causa esse erro; causaria erro só se você chamasse um `Method` como se fosse `ClassMethod`.

---

## 9. Resumo relâmpago

1. **IRIS** é banco de dados, linguagem e servidor de aplicação numa coisa só; fica ligado como serviço.
2. **Namespace** é a sala onde seu código e seus dados vivem; `USER` é rascunho, `%SYS` é administração, `LABSTUDY` é o nosso.
3. **Database** é o armário físico (`IRIS.DAT`); **namespace** é o nome que aponta para ele.
4. **Terminal** é o balcão de comandos; o prompt mostra o namespace atual.
5. `ZN "NOME"` troca de namespace — **com aspas**. `HALT` encerra a sessão.
6. **Classe** é o molde; **objeto** é a peça feita nesse molde. Classes ficam em arquivos `.cls`.
7. **Salvar não é compilar.** Sem compilar, o IRIS não conhece o seu código.
8. Um espaço entre comando e argumento. **Dois espaços significam "sem argumento"** e quebram tudo.
9. Comandos e funções **não** diferenciam maiúsculas; nomes de variáveis, classes e métodos **diferenciam**.
10. `$` = coisa do sistema (função ou variável especial). `$$$` = macro.
11. `^` = global, ou seja, dado gravado em disco. Sem `^` = variável local, some ao fim da sessão.
12. `%` = item do sistema. Nunca crie nomes começando com `%`.
13. `#` = instrução para o compilador; `..#NOME` lê um parâmetro da própria classe.
14. `##class(Pacote.Classe).Metodo()` chama um método de classe de fora dela; `..Metodo()` chama de dentro.
15. `_` cola textos; `+` soma números. `WRITE` mostra o valor; `ZWRITE` mostra nome e valor.
16. `DO` executa e descarta o retorno; `WRITE` executa e mostra o retorno.
17. O caminho do arquivo espelha o pacote: `LabStudy.Model.Patient` → `src/LabStudy/Model/Patient.cls`.

---

## 10. Cartões de memorização

**Frente:** O que é um namespace?
**Verso:** A "sala" lógica que contém código e dados. Você trabalha em um por vez; o mesmo código em salas diferentes não se enxerga.

**Frente:** Qual a diferença entre namespace e database?
**Verso:** Database é o arquivo físico `IRIS.DAT`. Namespace é o nome lógico que aponta para um ou mais databases.

**Frente:** O que significa o `^` antes de um nome?
**Verso:** Global — variável gravada em disco, que sobrevive ao fim da sessão.

**Frente:** O que significa `$` antes de um nome?
**Verso:** Item do sistema: função (`$LENGTH(...)`) ou variável especial (`$NAMESPACE`).

**Frente:** O que significa `$$$`?
**Verso:** Macro — um apelido substituído por outro texto na hora da compilação. Ex.: `$$$OK`.

**Frente:** O que significa `%` no início de um nome?
**Verso:** Item pertencente ao sistema (`%String`, `%Persistent`, `%SYS`). Reservado; não use nos seus nomes.

**Frente:** Como se lê `##class(LabStudy.App).About()`?
**Verso:** "Da classe LabStudy.App, execute o método About."

**Frente:** Como se lê `..#VERSION`?
**Verso:** "O parâmetro VERSION da minha própria classe."

**Frente:** Quantos espaços vão entre um comando e seu argumento?
**Verso:** Exatamente um. Dois espaços significam "comando sem argumento" e causam `<SYNTAX>`.

**Frente:** `name` e `Name` são a mesma variável?
**Verso:** Não. Nomes de variáveis diferenciam maiúsculas de minúsculas.

**Frente:** Qual comando troca de namespace no Terminal?
**Verso:** `ZN "NOMEDANAMESPACE"` — com aspas duplas.

**Frente:** Qual comando encerra a sessão do Terminal?
**Verso:** `HALT`.

**Frente:** Qual a diferença entre `WRITE` e `ZWRITE`?
**Verso:** `WRITE` mostra só o valor; `ZWRITE` mostra nome e valor formatados, e percorre todos os subscritos de uma global.

**Frente:** O que faz o `!` dentro de um `WRITE`?
**Verso:** Pula para a próxima linha.

**Frente:** O que faz o `_` entre dois valores?
**Verso:** Concatena (cola) os dois como texto. `"20" _ "3"` = `"203"`.

**Frente:** Qual a diferença entre `DO` e `WRITE` ao chamar um método que devolve valor?
**Verso:** `DO` executa e descarta o retorno; `WRITE` executa e imprime o retorno.

**Frente:** Erro `<CLASS DOES NOT EXIST>` — o que verificar primeiro?
**Verso:** Se compilou, e em qual namespace compilou versus em qual está executando.

**Frente:** Erro `<METHOD DOES NOT EXIST>` — o que ele indica?
**Verso:** A classe existe compilada, mas o método procurado não está na versão compilada. Normalmente falta compilar de novo.

**Frente:** Onde fica o arquivo da classe `LabStudy.Model.Patient`?
**Verso:** `src/LabStudy/Model/Patient.cls` — cada ponto do pacote vira uma pasta.

**Frente:** O que representa `$$$OK`?
**Verso:** O `%Status` de sucesso. Seu valor é `1`.

---

Digite CONTINUAR para o próximo capítulo.
