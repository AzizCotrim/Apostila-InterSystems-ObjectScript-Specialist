# Apostila InterSystems ObjectScript Specialist
## Capítulo 7 — T2.3 Implements Security Features (Recursos de segurança)

> Último tópico do domínio **T2 — Basic Programming**. Aqui você aprende como o IRIS decide quem pode fazer o quê, como verificar isso a partir do seu código, e como proteger dados sensíveis dentro da aplicação.

---

## 1. Objetivo do capítulo

Ao terminar este capítulo, você será capaz de:

1. Explicar o modelo de segurança do IRIS: **usuário, papel, recurso, privilégio, serviço e aplicação**.
2. Ler o contexto de segurança do processo com **`$USERNAME`** e **`$ROLES`**.
3. Verificar privilégios a partir do código com **`$SYSTEM.Security.Check()`**.
4. Elevar privilégios de forma temporária e segura com **`NEW $ROLES`** e **`$SYSTEM.Security.AddRoles()`**.
5. Entender **papéis de aplicação** e por que eles são preferíveis a dar permissão permanente ao usuário.
6. Proteger dados sensíveis com **hash** (`SHAHash`) e **sal**, e entender por que hash **não é** criptografia.
7. Cifrar e decifrar informação com **`AESCBCEncrypt`** / **`AESCBCDecrypt`**, e codificar em texto com **Base64**.
8. Distinguir **criptografia no banco** (*data at rest*) de criptografia feita pela aplicação.
9. Entender e aplicar **segurança em nível de linha** (`ROWLEVELSECURITY`, `%SecurityPolicy`, `%READERLIST`).
10. Reconhecer e evitar **injeção de SQL**, usando parâmetros em vez de concatenação.
11. Aplicar o princípio do **menor privilégio** no desenho de uma aplicação.
12. Evoluir o projeto: autenticação própria com senha protegida, campo clínico cifrado e verificação de privilégio nas operações sensíveis.

---

## 2. O conceito em linguagem de gente

### 2.1 O modelo do IRIS, contado como um prédio

Imagine o prédio do laboratório. Para descrever quem entra onde, você precisa de quatro ideias, e o IRIS usa exatamente essas quatro, com esses nomes:

**1. Recurso (*resource*) — a porta.**
Cada coisa que precisa ser protegida é um recurso: a sala do arquivo, a sala do servidor, a sala da diretoria. No IRIS, um recurso é qualquer bem protegido: um banco de dados, um serviço, uma aplicação, ou algo que você mesmo inventou.

Recursos de banco de dados têm nome padronizado começando com `%DB_`. O banco `LABSTUDYDATA` que criamos no Capítulo 0 tem o recurso `%DB_LABSTUDYDATA`.

**2. Privilégio (*privilege*) — a permissão numa porta específica.**
Não basta dizer "acesso à sala do arquivo". É preciso dizer **o que** a pessoa pode fazer lá: só olhar, ou também mexer. Um privilégio é sempre a soma de **recurso + permissão**.

As permissões são três:

- **`READ`** (`R`) — ler.
- **`WRITE`** (`W`) — alterar.
- **`USE`** (`U`) — usar (aplicável a serviços e aplicações, não a bancos).

Escreve-se assim: `%DB_LABSTUDYDATA:READ`, `%Development:USE`.

**3. Papel (*role*) — o molho de chaves.**
Ninguém sai distribuindo chave por chave. Você monta molhos: o molho "recepção", o molho "técnico de laboratório", o molho "administrador". Cada molho é um conjunto de privilégios.

Um **papel** é um molho de chaves. O IRIS já traz alguns prontos, sendo o mais poderoso o **`%All`**, que abre todas as portas — o equivalente ao molho do síndico.

**4. Usuário (*user*) — a pessoa.**
Cada pessoa recebe um ou mais molhos. A pessoa não recebe chaves avulsas; recebe molhos.

A cadeia completa, então, é:

```
usuário  ->  papéis  ->  privilégios  ->  recursos
```

E a pergunta que o IRIS faz a cada operação é sempre a mesma: *"algum dos papéis deste usuário contém um privilégio que dá esta permissão sobre este recurso?"*

Além disso existem mais duas peças:

**5. Serviço (*service*) — a portaria de entrada.**
Um serviço é uma forma de entrar no sistema: pela porta web, pelo terminal, por uma conexão de banco. Cada uma pode ser ligada, desligada e restringida separadamente.

**6. Aplicação (*application*) — a área do prédio com regra própria.**
Uma aplicação (por exemplo, uma aplicação web) pode ter **papéis próprios**, concedidos apenas enquanto o usuário está dentro dela. É a ideia do crachá de visitante: dentro daquela área, você pode mais; ao sair, o crachá é recolhido.

### 2.2 O princípio que organiza tudo: menor privilégio

A regra profissional que você deve carregar para a vida:

> **Dê a cada usuário e a cada processo exatamente o mínimo necessário para fazer o trabalho, e nada além.**

Isso soa óbvio e quase ninguém segue. A tentação é sempre a mesma: dar `%All` para "resolver logo" e prometer ajustar depois. O ajuste nunca vem.

O antídoto prático é a **elevação temporária**: em vez de o usuário ter permissão permanente para uma operação perigosa, ele tem permissão zero, e o **código** eleva o privilégio apenas durante a operação específica, devolvendo-o em seguida. Vamos ver exatamente como fazer isso na seção 3.4.

### 2.3 Hash não é criptografia

Esta distinção é o conceito mais importante do capítulo do ponto de vista prático, e cai na prova.

**Criptografia é um cofre com chave.** Você guarda o documento dentro, tranca, e depois — com a chave — abre e recupera o documento **exatamente como era**. É um caminho de **ida e volta**.

**Hash é um moedor de carne.** Você põe o documento, ele sai transformado numa massa de tamanho fixo. Não existe caminho de volta. É **só ida**.

Se hash não permite recuperar o original, para que serve?

Serve para **comparar sem guardar**. Você não precisa saber a senha de ninguém; precisa apenas saber se a senha digitada é a mesma de antes. Então você guarda a massa moída. Quando a pessoa digita a senha, você mói de novo e compara as duas massas. Se batem, a senha está certa — e você nunca teve a senha guardada em lugar nenhum.

Regra de decisão, definitiva:

| Preciso... | Uso |
|---|---|
| verificar uma senha | **hash** (não guarde a senha, nunca) |
| guardar um dado que precisarei ler de volta (um número de documento, um laudo) | **criptografia** |
| garantir que um arquivo não foi alterado | **hash** |
| transmitir um segredo e recuperá-lo depois | **criptografia** |

Guardar senha cifrada em vez de com hash é um erro sério: quem obtiver a chave obtém todas as senhas. Guardar senha em texto puro é um erro grave e indefensável.

### 2.4 O sal

Um problema do hash puro: se duas pessoas usam a mesma senha, as duas massas moídas são **idênticas**. Um atacante que veja a tabela percebe imediatamente quem compartilha senha — e pode usar tabelas prontas de senhas comuns já moídas para descobrir quais são.

A solução é o **sal** (*salt*): um pedaço aleatório, diferente para cada usuário, que é misturado à senha antes de moer.

Analogia: você tempera cada carne com uma mistura diferente antes de moer. Mesmo que duas pessoas usem a mesma carne, as massas saem diferentes.

O sal **não é secreto**: ele fica guardado ao lado do hash, em texto claro. Isso é intencional e correto — você precisa dele para reproduzir o cálculo na hora de conferir. O que ele impede é a comparação em massa e o uso de tabelas pré-calculadas.

### 2.5 Segurança em nível de linha

Até aqui, a permissão foi "pode ou não pode acessar esta tabela". Às vezes você precisa de algo mais fino: *"cada médico enxerga apenas os pacientes dele"*.

O IRIS oferece isso nativamente: cada **linha** carrega a lista de papéis que podem lê-la. Quem não tiver nenhum desses papéis simplesmente **não vê a linha** — ela não aparece no resultado da consulta, como se não existisse.

Analogia: em vez de trancar a sala do arquivo, cada ficha tem uma etiqueta dizendo quem pode lê-la, e o arquivista entrega apenas as fichas que o solicitante pode ver. Ele não diz "há mais fichas que você não pode ver". Ele simplesmente entrega as permitidas.

### 2.6 Injeção de SQL

Um último conceito, que é a vulnerabilidade mais comum em aplicações de banco de dados.

Imagine que você monta uma consulta colando texto:

```
"SELECT * FROM LabStudy.PATIENT WHERE Name = '" _ nomeDigitado _ "'"
```

Se o usuário digitar `Maria`, sai:

```sql
SELECT * FROM LabStudy.PATIENT WHERE Name = 'Maria'
```

Mas se ele digitar `' OR 1=1 --`, sai:

```sql
SELECT * FROM LabStudy.PATIENT WHERE Name = '' OR 1=1 --'
```

E isso devolve **todos os pacientes**. O usuário não preencheu um campo: ele **reescreveu a sua consulta**.

Analogia: é como deixar um espaço em branco num contrato para o outro lado preencher, e ele escrever ali uma cláusula nova em vez de um nome.

A defesa é simples e absoluta: **nunca cole valores dentro do texto da consulta.** Use **parâmetros** — marcados com `?` — e passe os valores separadamente. Assim, o valor é sempre tratado como valor, nunca como pedaço de comando, não importa o que ele contenha.

Vamos ver a sintaxe completa disso no Capítulo 15, quando tratarmos de SQL a partir do ObjectScript. Por ora, grave a regra.

---

## 3. A sintaxe explicada

### 3.1 Lendo o contexto de segurança

```
LABSTUDY>WRITE $USERNAME, !
_SYSTEM

LABSTUDY>WRITE $ROLES, !
%All

LABSTUDY>WRITE $NAMESPACE, !
LABSTUDY
```

- **`$USERNAME`** — o usuário autenticado neste processo.
- **`$ROLES`** — a lista de papéis ativos **agora**, separados por vírgula. Note que essa lista pode mudar durante a execução, quando há elevação de privilégio.

Há uma distinção que a prova cobra:

- **Papéis de login** — os que o usuário tem por definição, ao entrar.
- **Papéis adicionados** — concedidos durante a execução, por uma aplicação ou por código.

`$ROLES` mostra a soma dos dois, ou seja, o que vale **neste instante**.

### 3.2 Verificando um privilégio no código

```objectscript
if $SYSTEM.Security.Check("%DB_LABSTUDYDATA", "READ") {
    write "posso ler o banco", !
}
```

**`$SYSTEM.Security.Check(recurso, permissão)`** devolve `1` se o processo atual tem aquele privilégio e `0` se não tem.

A permissão pode ser escrita por extenso (`"READ"`, `"WRITE"`, `"USE"`) ou pela inicial (`"R"`, `"W"`, `"U"`). Pode-se também verificar mais de uma de uma vez, como `"READ,WRITE"`.

Este é o comando central do capítulo do ponto de vista do desenvolvedor: é assim que o **seu** código pergunta ao IRIS se pode prosseguir.

Um padrão profissional de guarda no início de um método:

```objectscript
ClassMethod DeletePatient(id As %String) As %Status
{
    if '$SYSTEM.Security.Check("LabStudy_Delete", "USE") {
        quit $$$ERROR($$$GeneralError, "You are not allowed to delete patients")
    }

    quit ##class(LabStudy.Patient).%DeleteId(id)
}
```

Repare: o recurso `LabStudy_Delete` é **inventado por você**. Recursos de aplicação são criados no Portal, em **System Administration → Security → Resources**, e depois incluídos nos papéis apropriados. O nome é livre; a convenção usual é prefixar com o nome do sistema.

### 3.3 Autenticando um usuário no código

```objectscript
set sc = $SYSTEM.Security.Login(username, password)
if $$$ISERR(sc) {
    quit $$$ERROR($$$GeneralError, "Invalid credentials")
}
```

**`$SYSTEM.Security.Login(usuário, senha)`** autentica e, se der certo, **troca o contexto de segurança do processo** para aquele usuário: `$USERNAME` e `$ROLES` passam a refletir o novo usuário.

Isso é usado por aplicações que fazem sua própria tela de login sobre a base de usuários do IRIS.

Existem restrições sobre quem pode chamar esse método e em que circunstâncias: **verificar na documentação oficial** antes de usar em produção.

### 3.4 Elevação temporária: o padrão `NEW $ROLES`

Este é o padrão mais elegante do capítulo, e o mais cobrado.

O problema: uma operação específica precisa de um privilégio que o usuário não deve ter permanentemente. Por exemplo, apagar registros antigos numa rotina de manutenção.

A solução errada: dar o papel ao usuário. Ele passa a poder apagar a qualquer momento, por qualquer caminho.

A solução certa: **o código empresta o papel, faz a operação e devolve.**

```objectscript
ClassMethod PurgeOldRecords() As %Status
{
    new $ROLES                                   // guarda os papéis atuais

    set sc = $SYSTEM.Security.AddRoles("LabStudyMaintenance")
    if $$$ISERR(sc) {
        quit sc
    }

    // ... a operação privilegiada acontece aqui ...

    quit $$$OK
}                                                // ao sair, $ROLES é restaurado
```

Como funciona, peça por peça:

- **`new $ROLES`** — o comando `NEW`, que você viu no Capítulo 3 aplicado a variáveis, funciona também sobre `$ROLES`. Ele **guarda o valor atual** e garante que, **quando o bloco terminar — por qualquer caminho, inclusive por erro — o valor original seja restaurado**.
- **`$SYSTEM.Security.AddRoles(lista)`** — acrescenta papéis ao processo. Devolve `%Status`.
- Ao sair do método, os papéis extras somem sozinhos.

A garantia importante é essa última: **a elevação não vaza**. Mesmo que o método termine por uma exceção, o `NEW $ROLES` restaura. Sem ele, um erro no meio deixaria o processo com privilégios elevados indefinidamente — uma falha de segurança séria.

Existe também **`$SYSTEM.Security.RemoveRoles(lista)`**, para retirar papéis explicitamente.

Um detalhe fundamental: `AddRoles` **não é um cheque em branco**. Só é possível acrescentar papéis que o usuário está autorizado a receber, conforme a configuração feita pelo administrador. O código não pode se autopromover a `%All`.

### 3.5 Hash: `SHAHash`

```objectscript
set hash = $SYSTEM.Encryption.SHAHash(256, "minha senha")
set readable = $SYSTEM.Encryption.Base64Encode(hash)
write readable, !
```

- **`$SYSTEM.Encryption.SHAHash(bits, texto)`** — calcula o hash. O primeiro argumento é o tamanho em bits: `1`, `224`, `256`, `384` ou `512`. **Use 256 ou mais**; hashes menores são considerados fracos hoje.
- O resultado é **binário**, cheio de caracteres não imprimíveis. Para guardar num campo de texto ou mostrar na tela, converta com **`Base64Encode`**.
- **`$SYSTEM.Encryption.Base64Decode(texto)`** faz o caminho de volta.

Base64 merece um aviso, porque é fonte de confusão perigosa: **Base64 não é segurança.** É apenas uma forma de representar bytes usando letras e números. Qualquer pessoa decodifica em um segundo. Ele serve para **transportar e armazenar** binário em campos de texto, não para esconder nada.

### 3.6 Gerando um sal

```objectscript
set salt = $SYSTEM.Encryption.Base64Encode($SYSTEM.Encryption.GenCryptRand(16))
```

**`$SYSTEM.Encryption.GenCryptRand(n)`** gera `n` bytes aleatórios de qualidade criptográfica. Isso é diferente de um número aleatório comum (`$RANDOM`), que é previsível e **não serve** para fins de segurança.

O sal é guardado junto do hash, em texto claro, e usado para conferir depois.

### 3.7 Criptografia simétrica: AES

```objectscript
set key = $SYSTEM.Encryption.GenCryptRand(32)      // chave de 32 bytes
set iv  = $SYSTEM.Encryption.GenCryptRand(16)      // vetor de inicialização

set cipher = $SYSTEM.Encryption.AESCBCEncrypt("dado sensivel", key, iv)
set plain  = $SYSTEM.Encryption.AESCBCDecrypt(cipher, key, iv)

write plain, !
```

- **`AESCBCEncrypt(texto, chave, iv)`** cifra; **`AESCBCDecrypt(cifra, chave, iv)`** decifra.
- A **chave** deve ter um comprimento válido para AES (16, 24 ou 32 bytes). O comprimento determina a força.
- O **IV** (*initialization vector*) é um valor de 16 bytes que faz com que o mesmo texto, cifrado duas vezes, produza resultados diferentes. Ele **não é secreto** e é guardado junto da cifra.
- O resultado é binário: use Base64 para armazenar em campo de texto.

Os requisitos exatos de comprimento e as variantes disponíveis (há também funções com gestão de chave pelo próprio IRIS) mudam entre versões: **verificar na documentação oficial**.

**O problema difícil da criptografia não é cifrar — é guardar a chave.** Se a chave está no código-fonte, quem lê o código tem tudo. Se está numa tabela do mesmo banco, quem rouba o banco tem tudo. As soluções profissionais envolvem cofres de chaves externos ou o gerenciamento de chaves do próprio IRIS. Guarde essa consciência: **cifrar com a chave ao lado do dado protege muito pouco.**

### 3.8 Criptografia do banco (*data at rest*)

Diferente de tudo acima, o IRIS pode cifrar **o arquivo de banco inteiro**, de forma transparente. Isso se configura na administração, não no código: o desenvolvedor não muda uma linha, e todo o `IRIS.DAT` fica ilegível para quem obtiver o arquivo.

A distinção que a prova cobra:

| Tipo | Protege contra | Quem configura |
|---|---|---|
| **Criptografia do banco** (*at rest*) | roubo do arquivo, do disco, do backup | administrador |
| **Criptografia pela aplicação** | leitura por quem tem acesso legítimo ao banco mas não deveria ver aquele campo | desenvolvedor |

As duas são complementares. A do banco não protege um campo contra um usuário que tem acesso à tabela; a da aplicação não protege nada se o atacante levar o disco e você tiver deixado a chave junto.

### 3.9 Segurança em nível de linha

```objectscript
Class LabStudy.Demo.Secret Extends %Persistent
{

Parameter ROWLEVELSECURITY = 1;

Property Title As %String(MAXLEN = 100);

Property OwnerRole As %String(MAXLEN = 64);

/// Returns the list of roles allowed to read this row.
ClassMethod %SecurityPolicy(Title As %String, OwnerRole As %String) As %String [ SqlProc ]
{
    quit $LISTBUILD(OwnerRole)
}

}
```

Como funciona:

- **`Parameter ROWLEVELSECURITY = 1`** liga o recurso.
- **`%SecurityPolicy`** é um `ClassMethod` de nome fixo que recebe os valores das propriedades da linha e devolve uma **lista** (`$LISTBUILD`) com os nomes dos papéis que podem lê-la.
- O IRIS grava essa lista numa propriedade especial chamada **`%READERLIST`**, criada automaticamente, e a consulta a cada leitura.
- Linhas cuja lista não bate com nenhum papel do usuário **não aparecem** nos resultados.

Cuidados que valem ponto:

- A lista é calculada **na gravação**, não na leitura. Se a política mudar, as linhas antigas continuam com a lista antiga até serem regravadas.
- Ligar `ROWLEVELSECURITY` numa classe que já tem dados exige regravar ou reconstruir para popular o `%READERLIST`.
- Usuários com `%All` normalmente enxergam tudo, o que dificulta testar o recurso: teste com um usuário comum.

A assinatura exata de `%SecurityPolicy` e as opções de configuração variam por versão: **verificar na documentação oficial**.

---

## 4. Exemplo comentado

Uma classe que reúne autenticação com hash, verificação de privilégio e campo cifrado:

Arquivo `src/LabStudy/Demo/Vault.cls`:

```objectscript
/// Demonstrates hashing, salting, symmetric encryption and privilege checks.
Class LabStudy.Demo.Vault Extends %Persistent
{

/// Number of bits used by the hash function.
Parameter HASHBITS = 256;

/// Resource that guards the sensitive operations of this class.
Parameter RESOURCE = "%DB_LABSTUDYDATA";

Property UserName As %String(MAXLEN = 64) [ Required ];

/// Random value stored in clear, different for every user.
Property Salt As %String(MAXLEN = 64);

/// Hash of salt + password. The password itself is never stored.
Property PasswordHash As %String(MAXLEN = 128);

/// A sensitive note, stored encrypted and Base64 encoded.
Property SecretNote As %String(MAXLEN = 500);

/// Builds the hash of a password using a given salt.
ClassMethod BuildHash(salt As %String, password As %String) As %String
{
    set raw = $SYSTEM.Encryption.SHAHash(..#HASHBITS, salt_password)
    quit $SYSTEM.Encryption.Base64Encode(raw)
}

/// Creates a user with a protected password.
ClassMethod Register(userName As %String, password As %String) As %String
{
    if $LENGTH(password) < 8 {
        write "Password must have at least 8 characters", !
        quit ""
    }

    set entry = ..%New()
    set entry.UserName = userName
    set entry.Salt = $SYSTEM.Encryption.Base64Encode($SYSTEM.Encryption.GenCryptRand(16))
    set entry.PasswordHash = ..BuildHash(entry.Salt, password)

    set sc = entry.%Save()
    if $$$ISERR(sc) {
        do $SYSTEM.Status.DisplayError(sc)
        quit ""
    }

    quit entry.%Id()
}

/// Checks a password without ever storing or comparing it in clear.
ClassMethod Verify(id As %String, password As %String) As %Boolean
{
    set entry = ..%OpenId(id)
    if '$ISOBJECT(entry) {
        quit 0
    }

    set candidate = ..BuildHash(entry.Salt, password)
    quit (candidate = entry.PasswordHash)
}

/// Stores an encrypted note. The key is supplied by the caller.
Method SetNote(text As %String, key As %String) As %Status
{
    set iv = $SYSTEM.Encryption.GenCryptRand(16)
    set cipher = $SYSTEM.Encryption.AESCBCEncrypt(text, key, iv)

    // The IV is not secret and travels with the ciphertext.
    set ..SecretNote = $SYSTEM.Encryption.Base64Encode(iv)_"|"_$SYSTEM.Encryption.Base64Encode(cipher)
    quit $$$OK
}

/// Reads back the encrypted note, if the caller has the privilege and the key.
Method GetNote(key As %String) As %String
{
    if '$SYSTEM.Security.Check(..#RESOURCE, "READ") {
        quit "(access denied)"
    }

    if ..SecretNote = "" {
        quit ""
    }

    set iv     = $SYSTEM.Encryption.Base64Decode($PIECE(..SecretNote, "|", 1))
    set cipher = $SYSTEM.Encryption.Base64Decode($PIECE(..SecretNote, "|", 2))

    quit $SYSTEM.Encryption.AESCBCDecrypt(cipher, key, iv)
}

/// Shows the current security context of the process.
ClassMethod WhoAmI() As %Status
{
    write "User:      ", $USERNAME, !
    write "Roles:     ", $ROLES, !
    write "Namespace: ", $NAMESPACE, !
    write "Can read ", ..#RESOURCE, ": ", $SYSTEM.Security.Check(..#RESOURCE, "READ"), !
    write "Can write ", ..#RESOURCE, ": ", $SYSTEM.Security.Check(..#RESOURCE, "WRITE"), !
    quit $$$OK
}

}
```

Comentando as decisões:

- **A senha nunca aparece numa propriedade.** Só existem `Salt` e `PasswordHash`. Mesmo com acesso total ao banco, ninguém lê senha de ninguém. Esse é o objetivo inteiro do desenho.
- **`BuildHash` é um método separado**, usado tanto no cadastro quanto na verificação. Se as duas rotinas calculassem o hash de formas ligeiramente diferentes, ninguém conseguiria entrar — e o bug seria difícil de achar. Um único ponto de cálculo elimina essa classe de erro.
- **O sal é gerado com `GenCryptRand`, não com `$RANDOM`.** Sal previsível não é sal.
- **O sal fica em texto claro**, e isso é correto. Ele não é segredo; é diversificador.
- **`Verify` compara hashes, nunca senhas.** E devolve simplesmente `0` quando o registro não existe, sem distinguir "usuário inexistente" de "senha errada" — porque distinguir os dois entrega informação a quem está tentando adivinhar.
- **O IV é gerado a cada gravação e guardado junto da cifra**, separado por `|`. Isso é a prática padrão: IV não é secreto, mas precisa ser diferente a cada operação.
- **A chave é recebida como argumento, não guardada na classe.** Isso empurra o problema difícil — onde guardar a chave — para fora, que é onde ele deve ser resolvido, com a infraestrutura adequada. Uma classe que guarda a própria chave ao lado dos dados cifrados está apenas fingindo proteger.
- **`GetNote` verifica privilégio antes de decifrar.** Duas barreiras independentes: você precisa **poder** e precisa **ter a chave**.
- **`$PIECE(texto, "|", n)`** pega o n-ésimo pedaço de um texto separado por `|`. Funções de string são o Capítulo 10.

### 4.1 Usando no Terminal

```
LABSTUDY>DO ##class(LabStudy.Demo.Vault).WhoAmI()
User:      _SYSTEM
Roles:     %All
Namespace: LABSTUDY
Can read %DB_LABSTUDYDATA: 1
Can write %DB_LABSTUDYDATA: 1

LABSTUDY>SET id = ##class(LabStudy.Demo.Vault).Register("recepcao02","segredo123")

LABSTUDY>SET v = ##class(LabStudy.Demo.Vault).%OpenId(id)

LABSTUDY>WRITE v.Salt, !
7Kq3xJm2Rf8pN1Ye4Vc0Bg==

LABSTUDY>WRITE v.PasswordHash, !
Uk9tHhZ2mQb4Xn8vLd1FpC5wA7sYgTz0Ie6RjKu3NcM=

LABSTUDY>WRITE ##class(LabStudy.Demo.Vault).Verify(id, "errada"), !
0

LABSTUDY>WRITE ##class(LabStudy.Demo.Vault).Verify(id, "segredo123"), !
1

LABSTUDY>SET key = $SYSTEM.Encryption.GenCryptRand(32)

LABSTUDY>DO v.SetNote("Patient has a rare condition", key)

LABSTUDY>WRITE v.SecretNote, !
9pQ2Lm4Xz8Vn1Kd0==|Rt7Yb3Nx1Qw9Ze5Ma2Cf8Vg4Jh6Kp0Ld3Sn7Bt1Xr5==

LABSTUDY>WRITE v.GetNote(key), !
Patient has a rare condition

LABSTUDY>SET wrongKey = $SYSTEM.Encryption.GenCryptRand(32)

LABSTUDY>WRITE v.GetNote(wrongKey), !
   (sai lixo ilegível ou um erro — a chave errada não decifra)
```

*(Os valores de sal, hash e cifra da sua execução serão diferentes: são aleatórios por construção.)*

O que observar:

- **O mesmo usuário, cadastrado duas vezes com a mesma senha, produziria hashes diferentes**, porque o sal é diferente. Faça esse teste; é a demonstração do que o sal faz.
- **`SecretNote` guardado é ilegível**, e nem o administrador do banco consegue lê-lo sem a chave.
- **Com a chave errada, não sai o texto.** Não há "quase certo" em criptografia.

### 4.2 A elevação temporária em ação

```objectscript
ClassMethod ShowRoleEscalation() As %Status
{
    write "before: ", $ROLES, !

    do ..InnerElevated()

    write "after:  ", $ROLES, !
    quit $$$OK
}

ClassMethod InnerElevated() As %Status
{
    new $ROLES
    set sc = $SYSTEM.Security.AddRoles("%Developer")
    write "inside: ", $ROLES, !
    quit $$$OK
}
```

```
LABSTUDY>DO ##class(LabStudy.Demo.Vault).ShowRoleEscalation()
before: %All
inside: %All,%Developer
after:  %All
```

O papel extra existiu **apenas** dentro do método interno. Ao sair, sumiu sozinho. Nenhuma linha de código teve que se lembrar de removê-lo — e é justamente por isso que o padrão é seguro: **ele não depende de ninguém lembrar.**

---

## 5. Variações e detalhes

### 5.1 Administrando usuários e papéis pelo código

As classes de administração de segurança vivem no namespace **`%SYS`**:

```
LABSTUDY>ZN "%SYS"

%SYS>SET sc = ##class(Security.Roles).Get("%Developer", .props)

%SYS>ZWRITE props
```

Existem `Security.Users`, `Security.Roles`, `Security.Resources`, `Security.Services` e `Security.Applications`, com métodos como `Create`, `Get`, `Modify`, `Delete` e `Exists`.

Três avisos:

1. **Só funcionam em `%SYS`.** Chamá-las de outro namespace falha.
2. **Exigem privilégio administrativo.**
3. **Mexer nisso por código é operação delicada.** Um erro pode trancar você para fora do sistema. Em estudo, prefira o Portal; conheça as classes para reconhecê-las na prova e para automações bem testadas.

### 5.2 Papéis de aplicação

Uma aplicação (web ou de terminal) pode ser configurada com **papéis próprios**, que valem só enquanto o código daquela aplicação está executando.

Isso é o crachá de visitante da seção 2.1, e é o mecanismo preferido em sistemas bem desenhados:

- O usuário, sozinho, não tem permissão nenhuma sobre os bancos.
- A **aplicação** tem a permissão.
- O usuário só consegue tocar nos dados **através da aplicação**, que impõe as regras de negócio.

Se ele tentar conectar diretamente ao banco por outra ferramenta, não consegue nada. Essa é a diferença entre "o usuário pode ler a tabela" e "o usuário pode usar a tela que lê a tabela" — e ela é enorme.

### 5.3 Privilégios de SQL

Além dos recursos de banco, o SQL tem seu próprio nível de permissão, no padrão da linguagem:

```sql
GRANT SELECT, INSERT ON LabStudy.PATIENT TO LabTechnician
REVOKE DELETE ON LabStudy.PATIENT FROM LabTechnician
```

Isso controla o que cada papel pode fazer **por tabela e por operação**. É mais fino que o recurso `%DB_`, que é do banco inteiro.

Para verificar a partir do SQL existe `%CHECKPRIV`, e há métodos correspondentes no pacote de SQL do sistema: **verificar na documentação oficial** para a forma exata.

### 5.4 Métodos e classes protegidos

Alguns controles ficam na própria definição:

- **`[ Private ]`** num método impede chamada de fora da classe. É encapsulamento, não segurança contra atacante — mas evita que uma operação interna seja invocada por engano.
- **`[ ServerOnly = 1 ]`** impede que o método seja exposto a clientes.
- Nunca conte apenas com "o método é privado" como barreira de segurança: quem controla o código controla tudo. Segurança de verdade se verifica com `$SYSTEM.Security.Check()`.

### 5.5 Onde colocar a verificação de privilégio

Regra prática: **na entrada da operação de negócio**, não espalhada.

- **Bom:** um `ClassMethod DeletePatient()` que verifica no início.
- **Ruim:** verificar dentro de um laço, dez vezes.
- **Pior:** verificar na tela e não verificar no método — porque a tela pode ser contornada.

E uma armadilha comum: verificar **depois** de já ter feito parte do trabalho. Verifique antes de qualquer efeito colateral.

### 5.6 Mensagens de erro que não entregam informação

Quando uma verificação falha, a mensagem deve dizer **o suficiente e nada mais**.

- **Ruim:** "Usuário `recepcao02` não tem o papel `LabStudyAdmin` no recurso `LabStudy_Delete`". Isso ensina ao atacante o nome do papel e do recurso.
- **Bom:** "Você não tem permissão para esta operação."

O detalhe completo deve ir para a **trilha de auditoria** do capítulo anterior, onde só quem investiga o vê. Essa combinação — mensagem discreta para o usuário, registro completo na trilha — é o padrão profissional.

---

## 6. Pegadinhas e erros comuns

**1) Guardar senha em texto puro.**
Erro grave e indefensável. Nunca.

**2) Guardar senha cifrada em vez de com hash.**
Cifra é reversível. Quem obtiver a chave obtém todas as senhas. Senha usa **hash**.

**3) Usar hash sem sal.**
Senhas iguais produzem hashes iguais, e tabelas pré-calculadas quebram as senhas comuns.

**4) Achar que o sal precisa ser secreto.**
Não precisa, e não pode: você precisa dele para conferir. Ele é diversificador, não segredo.

**5) Achar que Base64 protege alguma coisa.**
Base64 é representação, não proteção. Decodifica-se instantaneamente.

**6) Usar `$RANDOM` para gerar sal, chave ou token.**
`$RANDOM` é previsível. Use `$SYSTEM.Encryption.GenCryptRand()`.

**7) Guardar a chave de criptografia ao lado do dado cifrado.**
Protege quase nada. O problema difícil da criptografia é a guarda da chave.

**8) Reutilizar o mesmo IV para tudo.**
O IV deve ser diferente a cada operação de cifragem. Ele não é secreto, mas precisa ser único.

**9) Elevar privilégio sem `NEW $ROLES`.**
Se o método falhar no meio, o processo fica com privilégio elevado indefinidamente.

**10) Achar que `AddRoles` pode conceder qualquer papel.**
Só papéis que o administrador autorizou o usuário a receber.

**11) Confundir criptografia do banco com criptografia da aplicação.**
A primeira protege o arquivo roubado; a segunda protege o campo de quem tem acesso legítimo à tabela. São complementares.

**12) Ligar `ROWLEVELSECURITY` numa classe com dados e esperar efeito imediato.**
A lista de leitores é calculada na gravação. Dados antigos precisam ser regravados ou reconstruídos.

**13) Testar segurança em nível de linha logado como `%All`.**
Quem tem tudo vê tudo. Teste com um usuário comum.

**14) Montar SQL colando texto do usuário.**
Injeção de SQL. Use parâmetros com `?`.

**15) Verificar permissão só na interface.**
A interface pode ser contornada. A verificação vale no método que executa a operação.

**16) Mensagens de erro que revelam nomes de papéis e recursos.**
Discrição para o usuário; detalhe completo para a trilha de auditoria.

---

## 7. MÃO NA MASSA

---

### Exercício 7.1 — Explorando o contexto de segurança

**a) Enunciado:** Crie `LabStudy.Demo.SecInfo` com um `ClassMethod Report()` que imprima:

1. Usuário, papéis e namespace atuais.
2. Se o processo pode **ler** e **escrever** no recurso `%DB_LABSTUDYDATA`.
3. Se o processo tem `USE` sobre o recurso `%Development`.
4. A quantidade de papéis ativos.

**b) Dica:** `$ROLES` é uma lista separada por vírgulas. Para contar os pedaços, `$LENGTH(texto, ",")`.

**c) Como testar:** Rodando como `_SYSTEM`, os privilégios devem sair todos como `1`.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

`src/LabStudy/Demo/SecInfo.cls`:

```objectscript
/// Prints the security context of the current process.
Class LabStudy.Demo.SecInfo Extends %RegisteredObject
{

/// Resources this report checks.
Parameter DBRESOURCE = "%DB_LABSTUDYDATA";

ClassMethod Report() As %Status
{
    write "=============================", !
    write "User:      ", $USERNAME, !
    write "Roles:     ", $ROLES, !
    write "Namespace: ", $NAMESPACE, !
    write "Job:       ", $JOB, !
    write "-----------------------------", !

    write ..#DBRESOURCE, " READ:  ", $SYSTEM.Security.Check(..#DBRESOURCE, "READ"), !
    write ..#DBRESOURCE, " WRITE: ", $SYSTEM.Security.Check(..#DBRESOURCE, "WRITE"), !
    write "%Development USE:      ", $SYSTEM.Security.Check("%Development", "USE"), !
    write "-----------------------------", !

    set count = $LENGTH($ROLES, ",")
    write "Active roles: ", count, !
    for i = 1:1:count {
        write "  ", i, ": ", $PIECE($ROLES, ",", i), !
    }
    write "=============================", !
    quit $$$OK
}

}
```

```
LABSTUDY>DO ##class(LabStudy.Demo.SecInfo).Report()
=============================
User:      _SYSTEM
Roles:     %All
Namespace: LABSTUDY
Job:       12345
-----------------------------
%DB_LABSTUDYDATA READ:  1
%DB_LABSTUDYDATA WRITE: 1
%Development USE:      1
-----------------------------
Active roles: 1
  1: %All
=============================
```

**Por que cada decisão:**

- **O nome do recurso está num `Parameter`**, não repetido cinco vezes no código. Nome de recurso muda quando o banco é renomeado; concentrar num lugar evita caçada.
- **A verificação é feita separadamente para `READ` e `WRITE`.** É perfeitamente possível ter uma e não a outra — na verdade, esse é o caso mais comum num sistema bem configurado, onde a maioria dos usuários lê e poucos escrevem.
- **`_SYSTEM` tem apenas o papel `%All`**, e isso basta para tudo. É exatamente por isso que ele é um péssimo usuário para testar segurança: com ele, nenhuma verificação jamais falha. Se você quiser ver uma verificação devolver `0`, precisa criar um usuário comum no Portal.
- **`$PIECE($ROLES, ",", i)`** separa a lista. Vale notar que essa lista é um texto simples separado por vírgulas, não uma estrutura — o que a torna fácil de inspecionar e fácil de errar se algum nome contiver vírgula.

---

### Exercício 7.2 — Elevação temporária de privilégio

**a) Enunciado:** Crie `LabStudy.Demo.Escalation` com:

1. `ClassMethod Outer()` — imprime `$ROLES`, chama `Elevated()`, imprime `$ROLES` de novo, chama `Leaky()` e imprime `$ROLES` uma terceira vez.
2. `ClassMethod Elevated()` — usa `new $ROLES`, acrescenta um papel, imprime `$ROLES`.
3. `ClassMethod Leaky()` — acrescenta o **mesmo** papel **sem** `new $ROLES`, imprime `$ROLES`.

Compare os três momentos e explique com suas palavras a diferença.

**b) Dica:** Use um papel que exista na sua instalação, como `%Developer`. Se `AddRoles` falhar, mostre o erro.

**c) Como testar:** Depois de `Elevated()`, os papéis devem voltar ao original. Depois de `Leaky()`, **não** devem.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

`src/LabStudy/Demo/Escalation.cls`:

```objectscript
/// Shows why NEW $ROLES is mandatory when elevating privileges.
Class LabStudy.Demo.Escalation Extends %RegisteredObject
{

/// Role temporarily borrowed in this demonstration.
Parameter EXTRAROLE = "%Developer";

ClassMethod Outer() As %Status
{
    write "1. before any call:      ", $ROLES, !

    do ..Elevated()
    write "2. after Elevated():     ", $ROLES, !

    do ..Leaky()
    write "3. after Leaky():        ", $ROLES, !

    quit $$$OK
}

/// The correct pattern: the extra role disappears on exit.
ClassMethod Elevated() As %Status
{
    new $ROLES

    set sc = $SYSTEM.Security.AddRoles(..#EXTRAROLE)
    if $$$ISERR(sc) {
        do $SYSTEM.Status.DisplayError(sc)
        quit sc
    }

    write "   inside Elevated():    ", $ROLES, !
    quit $$$OK
}

/// The wrong pattern: the extra role leaks to the caller.
ClassMethod Leaky() As %Status
{
    set sc = $SYSTEM.Security.AddRoles(..#EXTRAROLE)
    if $$$ISERR(sc) {
        do $SYSTEM.Status.DisplayError(sc)
        quit sc
    }

    write "   inside Leaky():       ", $ROLES, !
    quit $$$OK
}

}
```

```
LABSTUDY>DO ##class(LabStudy.Demo.Escalation).Outer()
1. before any call:      %All
   inside Elevated():    %All,%Developer
2. after Elevated():     %All
   inside Leaky():       %All,%Developer
3. after Leaky():        %All,%Developer
```

**Por que cada decisão:**

- **A comparação lado a lado é o exercício inteiro.** Os dois métodos fazem exatamente a mesma coisa; a única diferença é uma linha. E o efeito é a diferença entre uma elevação controlada e um vazamento permanente de privilégio no processo.
- **`new $ROLES` protege contra saídas inesperadas.** Acrescente propositalmente um erro no meio do `Elevated()` — por exemplo, uma divisão por zero — e observe: mesmo com o método explodindo, os papéis voltam. Faça esse teste; ele é convincente.
- **Não existe um `RemoveRoles` no `Elevated()`**, e isso é intencional. Se a limpeza dependesse de uma linha no final, qualquer `return` antecipado ou erro pularia essa linha. O `NEW` não pode ser pulado.
- **O papel está num `Parameter`.** Nome de papel é configuração, não literal espalhado.
- **Depois de rodar o exercício, o seu processo ficou com um papel a mais.** Feche e reabra o Terminal para voltar ao normal. Esse incômodo é a lição: em produção, o "a mais" seria uma brecha aberta pelo resto da sessão.

---

### Exercício 7.3 — Senha protegida com hash e sal

**a) Enunciado:** Crie `LabStudy.Demo.Account2`, persistente, com `Login As %String(MAXLEN = 64)` (com índice único), `Salt As %String(MAXLEN = 64)` e `PasswordHash As %String(MAXLEN = 128)`. Escreva:

1. `ClassMethod Create(login, password)` — recusa senha com menos de 8 caracteres, gera sal, calcula o hash e grava.
2. `ClassMethod Authenticate(login, password) As %Boolean` — localiza pelo índice único e confere.
3. `ClassMethod ChangePassword(login, oldPassword, newPassword) As %Status` — só troca se a senha antiga conferir, e gera um **sal novo**.

Depois, no Terminal: crie **dois** usuários com **exatamente a mesma senha** e mostre que os hashes são diferentes.

**b) Dica:** Use o método gerado pelo índice único (`LoginIdxOpen`) para a busca. Concentre o cálculo do hash num único método privado.

**c) Como testar:** `Authenticate` deve devolver `1` só com a senha correta. Os dois usuários com a mesma senha devem ter hashes diferentes.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

`src/LabStudy/Demo/Account2.cls`:

```objectscript
/// Stores credentials safely: salted hashes, never the password itself.
Class LabStudy.Demo.Account2 Extends %Persistent
{

Parameter HASHBITS = 256;

Parameter SALTBYTES = 16;

Parameter MINPASSWORD = 8;

Property Login As %String(MAXLEN = 64) [ Required ];

Property Salt As %String(MAXLEN = 64);

Property PasswordHash As %String(MAXLEN = 128);

/// Logins are unique; gives us LoginIdxOpen for free.
Index LoginIdx On Login [ Unique ];

/// Single point of hash calculation. Used by every other method.
ClassMethod Compute(salt As %String, password As %String) As %String [ Private ]
{
    quit $SYSTEM.Encryption.Base64Encode($SYSTEM.Encryption.SHAHash(..#HASHBITS, salt_password))
}

/// Generates a fresh random salt.
ClassMethod NewSalt() As %String [ Private ]
{
    quit $SYSTEM.Encryption.Base64Encode($SYSTEM.Encryption.GenCryptRand(..#SALTBYTES))
}

/// Creates an account. Returns the id, or "" on failure.
ClassMethod Create(login As %String, password As %String) As %String
{
    if $LENGTH(password) < ..#MINPASSWORD {
        write "Password must have at least ", ..#MINPASSWORD, " characters", !
        quit ""
    }

    set account = ..%New()
    set account.Login = login
    set account.Salt = ..NewSalt()
    set account.PasswordHash = ..Compute(account.Salt, password)

    set sc = account.%Save()
    if $$$ISERR(sc) {
        do $SYSTEM.Status.DisplayError(sc)
        quit ""
    }

    quit account.%Id()
}

/// Returns 1 when the password matches, 0 otherwise.
ClassMethod Authenticate(login As %String, password As %String) As %Boolean
{
    set account = ..LoginIdxOpen(login)
    if '$ISOBJECT(account) {
        quit 0
    }

    quit (..Compute(account.Salt, password) = account.PasswordHash)
}

/// Changes the password, generating a brand new salt.
ClassMethod ChangePassword(login As %String, oldPassword As %String, newPassword As %String) As %Status
{
    if '..Authenticate(login, oldPassword) {
        quit $$$ERROR($$$GeneralError, "Current password does not match")
    }

    if $LENGTH(newPassword) < ..#MINPASSWORD {
        quit $$$ERROR($$$GeneralError, "New password is too short")
    }

    set account = ..LoginIdxOpen(login)
    set account.Salt = ..NewSalt()
    set account.PasswordHash = ..Compute(account.Salt, newPassword)

    quit account.%Save()
}

}
```

```
LABSTUDY>DO ##class(LabStudy.Demo.Account2).%KillExtent()

LABSTUDY>WRITE ##class(LabStudy.Demo.Account2).Create("ana","curta"), !
Password must have at least 8 characters


LABSTUDY>WRITE ##class(LabStudy.Demo.Account2).Create("ana","mesmasenha1"), !
1

LABSTUDY>WRITE ##class(LabStudy.Demo.Account2).Create("bruno","mesmasenha1"), !
2

LABSTUDY>SET a = ##class(LabStudy.Demo.Account2).%OpenId(1)
LABSTUDY>SET b = ##class(LabStudy.Demo.Account2).%OpenId(2)

LABSTUDY>WRITE a.Salt, !
Xr4Kp9Mz2Vb7Nc1Qa5Ld8g==
LABSTUDY>WRITE b.Salt, !
7Yt3Jf0Ws6Ru2Hd9Km4Pn1Q==

LABSTUDY>WRITE a.PasswordHash, !
Qw8Zx1Nm5Vc9Bp3Ld7Ks2Ff6Gt0Yh4Ju8Ri2Oe6=
LABSTUDY>WRITE b.PasswordHash, !
Mn2Kd8Ry4Tz0Vb6Qs1Xw5Cj9Lp3Hf7Ng1Ae5Ui9=

LABSTUDY>WRITE (a.PasswordHash = b.PasswordHash), !
0

LABSTUDY>WRITE ##class(LabStudy.Demo.Account2).Authenticate("ana","errada"), !
0
LABSTUDY>WRITE ##class(LabStudy.Demo.Account2).Authenticate("ana","mesmasenha1"), !
1
LABSTUDY>WRITE ##class(LabStudy.Demo.Account2).Authenticate("naoexiste","qualquer"), !
0

LABSTUDY>WRITE $$$ISOK(##class(LabStudy.Demo.Account2).ChangePassword("ana","mesmasenha1","novasenha99")), !
1

LABSTUDY>WRITE ##class(LabStudy.Demo.Account2).Authenticate("ana","mesmasenha1"), !
0
LABSTUDY>WRITE ##class(LabStudy.Demo.Account2).Authenticate("ana","novasenha99"), !
1
```

**Por que cada decisão:**

- **A linha central do exercício é `(a.PasswordHash = b.PasswordHash)` devolvendo `0`.** Duas pessoas, a mesma senha, hashes completamente diferentes. Sem sal, seriam idênticos, e quem olhasse a tabela saberia disso na hora. É a demonstração mais direta do que o sal faz.
- **`Compute` e `NewSalt` são `[ Private ]`.** Ninguém de fora deve calcular hash por conta própria; o cálculo é assunto interno da classe.
- **`ChangePassword` gera um sal novo.** Reaproveitar o sal antigo funcionaria, mas trocar é melhor prática: reduz qualquer relação entre a credencial antiga e a nova.
- **`Authenticate` devolve `0` tanto para usuário inexistente quanto para senha errada.** Nenhuma dica é dada a quem tenta adivinhar.
- **O índice único sobre `Login`** faz dois trabalhos ao mesmo tempo: garante que não haja logins repetidos e entrega o `LoginIdxOpen`, que dispensa saber o ID interno.
- **Uma observação honesta de profundidade:** `SHAHash` é rápido — e velocidade é ruim para senhas, porque também acelera quem tenta adivinhar por força bruta. Sistemas de produção usam funções deliberadamente lentas, projetadas para senhas. Para a certificação, o que se cobra é hash + sal; para a vida real, saiba que existe esse degrau a mais.

---

### Exercício 7.4 — Cifrando e decifrando um campo

**a) Enunciado:** Crie `LabStudy.Demo.Envelope`, persistente, com `Label As %String(MAXLEN = 100)` e `Payload As %String(MAXLEN = 1000)`. Escreva:

1. `Method Seal(text, key)` — cifra o texto com AES, gerando um IV novo, e guarda `Base64(iv)|Base64(cifra)` em `Payload`.
2. `Method Open(key) As %String` — decifra e devolve o texto.
3. Prove, no Terminal, que: (a) o conteúdo gravado é ilegível; (b) cifrar o **mesmo texto duas vezes** produz resultados **diferentes**; (c) a chave errada não recupera o texto.

**b) Dica:** Chave de 32 bytes e IV de 16, ambos com `GenCryptRand`. Guarde a chave numa variável do Terminal para poder usá-la nos dois momentos.

**c) Como testar:** O item (b) é o que prova que o IV está sendo gerado a cada vez.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

`src/LabStudy/Demo/Envelope.cls`:

```objectscript
/// Stores a value encrypted with AES-CBC.
/// The IV is not secret and is stored next to the ciphertext.
Class LabStudy.Demo.Envelope Extends %Persistent
{

Parameter IVBYTES = 16;

Property Label As %String(MAXLEN = 100);

/// Format: Base64(iv) "|" Base64(ciphertext)
Property Payload As %String(MAXLEN = 1000);

/// Encrypts text into Payload, using a fresh IV every time.
Method Seal(text As %String, key As %String) As %Status
{
    set iv = $SYSTEM.Encryption.GenCryptRand(..#IVBYTES)
    set cipher = $SYSTEM.Encryption.AESCBCEncrypt(text, key, iv)

    set ..Payload = $SYSTEM.Encryption.Base64Encode(iv)
                    _"|"
                    _$SYSTEM.Encryption.Base64Encode(cipher)
    quit $$$OK
}

/// Decrypts Payload back into clear text.
Method Open(key As %String) As %String
{
    if ..Payload = "" {
        quit ""
    }

    set iv     = $SYSTEM.Encryption.Base64Decode($PIECE(..Payload, "|", 1))
    set cipher = $SYSTEM.Encryption.Base64Decode($PIECE(..Payload, "|", 2))

    quit $SYSTEM.Encryption.AESCBCDecrypt(cipher, key, iv)
}

}
```

```
LABSTUDY>SET key = $SYSTEM.Encryption.GenCryptRand(32)

LABSTUDY>SET e1 = ##class(LabStudy.Demo.Envelope).%New()
LABSTUDY>SET e1.Label = "note A"
LABSTUDY>DO e1.Seal("confidential result", key)
LABSTUDY>WRITE $$$ISOK(e1.%Save()), !
1

LABSTUDY>SET e2 = ##class(LabStudy.Demo.Envelope).%New()
LABSTUDY>SET e2.Label = "note B"
LABSTUDY>DO e2.Seal("confidential result", key)
LABSTUDY>WRITE $$$ISOK(e2.%Save()), !
1

LABSTUDY>WRITE e1.Payload, !
Kd8Zx2Mn4Vb1Qp==|R7Yt3Jf0Ws6Ru2Hd9Km4Pn1QaLd8Xr4Kp9Mz2Vb7Nc1Q==

LABSTUDY>WRITE e2.Payload, !
9Pn1QaLd8Xr4Kp==|Zx2Mn4Vb1QpR7Yt3Jf0Ws6Ru2Hd9Km4Kd8Mz2Vb7Nc1Q==

LABSTUDY>WRITE (e1.Payload = e2.Payload), !
0

LABSTUDY>WRITE e1.Open(key), !
confidential result

LABSTUDY>WRITE e2.Open(key), !
confidential result

LABSTUDY>SET wrong = $SYSTEM.Encryption.GenCryptRand(32)

LABSTUDY>WRITE e1.Open(wrong), !
   (lixo ilegível ou erro — a chave errada não recupera nada)
```

**Por que cada decisão:**

- **O item (b) é o coração do exercício.** Dois envelopes, o **mesmo texto**, a **mesma chave** — e conteúdos gravados **diferentes**. Isso é o IV trabalhando. Se o IV fosse fixo, os dois seriam idênticos, e um observador saberia que os dois registros guardam a mesma informação, sem precisar decifrar nada. Vazar "estes dois campos são iguais" já é vazar informação.
- **O IV vai junto, em claro, separado por `|`.** Isso não enfraquece nada. O que enfraqueceria seria repeti-lo.
- **A chave nunca é gravada.** Ela vive só na variável do Terminal, neste exercício. Num sistema real, ela viria de um cofre externo ou do gerenciamento de chaves do IRIS — nunca de uma constante no código nem de uma tabela do mesmo banco.
- **A concatenação foi quebrada em três linhas** com o `_` no início de cada continuação. Isso é apenas legibilidade; o ObjectScript aceita a expressão continuando na linha seguinte dentro do corpo do método.
- **`MAXLEN = 1000` no `Payload`** porque texto cifrado e codificado em Base64 cresce em relação ao original. Dimensionar campos cifrados pelo tamanho do texto claro é erro clássico.

---

### Exercício 7.5 — PROJETO CONTÍNUO: segurança no laboratório

**a) Enunciado:** Evolua o sistema:

1. Crie `LabStudy.User`, persistente, com `Login`, `FullName`, `Salt`, `PasswordHash`, `AppRole As %String(VALUELIST = ",reception,technician,admin")` e `Active As %Boolean [ InitialExpression = 1 ]`. Índice único em `Login`. Métodos:
   - `ClassMethod Create(login, fullName, password, appRole) As %String`
   - `ClassMethod Authenticate(login, password) As %Boolean`
   - `ClassMethod RoleOf(login) As %String`
2. Crie `LabStudy.Security` com:
   - `ClassMethod CanDelete() As %Boolean` — verifica `$SYSTEM.Security.Check` sobre o recurso do banco com permissão `WRITE`;
   - `ClassMethod Require(operation) As %Status` — devolve erro genérico se não puder, e **registra a tentativa negada na trilha de auditoria** do Capítulo 6;
   - `ClassMethod Context() As %Status` — imprime usuário, papéis e namespace.
3. Em `LabStudy.Exam`, acrescente `Property ClinicalNote As %String(MAXLEN = 1000)` guardando o texto **cifrado**, com `Method SetClinicalNote(text, key)` e `Method GetClinicalNote(key)`.
4. Em `LabStudy.Patient`, proteja a exclusão: crie `ClassMethod DeleteSecurely(id) As %Status`, que chama `LabStudy.Security.Require("DeletePatient")` antes de apagar.
5. Suba `LabStudy.App` para `"0.8"` e acrescente `ClassMethod SecurityReport()`.

**b) Dica:** Reaproveite integralmente o desenho de hash e sal do exercício 7.3 e o de cifragem do 7.4. A trilha de auditoria é a `LabStudy.AuditEntry` do capítulo anterior.

**c) Como testar:**

```
LABSTUDY>DO ##class(LabStudy.User).%KillExtent()
LABSTUDY>WRITE ##class(LabStudy.User).Create("recepcao02","Carla Nunes","senhaForte1","reception"), !
LABSTUDY>WRITE ##class(LabStudy.User).Authenticate("recepcao02","errada"), !
LABSTUDY>WRITE ##class(LabStudy.User).Authenticate("recepcao02","senhaForte1"), !
LABSTUDY>DO ##class(LabStudy.App).SecurityReport()
```

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

`src/LabStudy/User.cls`:

```objectscript
/// An application user of the LabStudy system.
/// The password is never stored: only a salted hash.
Class LabStudy.User Extends %Persistent [ SqlTableName = APP_USER ]
{

Parameter HASHBITS = 256;

Parameter SALTBYTES = 16;

Parameter MINPASSWORD = 8;

Property Login As %String(MAXLEN = 64) [ Required ];

Property FullName As %String(MAXLEN = 120);

Property Salt As %String(MAXLEN = 64);

Property PasswordHash As %String(MAXLEN = 128);

Property AppRole As %String(VALUELIST = ",reception,technician,admin");

Property Active As %Boolean [ InitialExpression = 1 ];

Index LoginIdx On Login [ Unique ];

/// Single point of hash calculation.
ClassMethod Compute(salt As %String, password As %String) As %String [ Private ]
{
    quit $SYSTEM.Encryption.Base64Encode($SYSTEM.Encryption.SHAHash(..#HASHBITS, salt_password))
}

ClassMethod NewSalt() As %String [ Private ]
{
    quit $SYSTEM.Encryption.Base64Encode($SYSTEM.Encryption.GenCryptRand(..#SALTBYTES))
}

/// Creates a user. Returns the id, or "" on failure.
ClassMethod Create(login As %String, fullName As %String, password As %String, appRole As %String = "reception") As %String
{
    if $LENGTH(password) < ..#MINPASSWORD {
        write "Password must have at least ", ..#MINPASSWORD, " characters", !
        quit ""
    }

    set user = ..%New()
    set user.Login = login
    set user.FullName = fullName
    set user.AppRole = appRole
    set user.Salt = ..NewSalt()
    set user.PasswordHash = ..Compute(user.Salt, password)

    set sc = user.%Save()
    if $$$ISERR(sc) {
        do $SYSTEM.Status.DisplayError(sc)
        quit ""
    }

    quit user.%Id()
}

/// Returns 1 only for an active user with the correct password.
ClassMethod Authenticate(login As %String, password As %String) As %Boolean
{
    set user = ..LoginIdxOpen(login)
    if '$ISOBJECT(user) {
        quit 0
    }
    if 'user.Active {
        quit 0
    }

    quit (..Compute(user.Salt, password) = user.PasswordHash)
}

/// Application role of a user, or "" when unknown.
ClassMethod RoleOf(login As %String) As %String
{
    set user = ..LoginIdxOpen(login)
    if '$ISOBJECT(user) {
        quit ""
    }
    quit user.AppRole
}

}
```

`src/LabStudy/Security.cls`:

```objectscript
/// Central place for privilege checks in the LabStudy system.
Class LabStudy.Security Extends %RegisteredObject
{

/// Resource that guards write access to laboratory data.
Parameter DBRESOURCE = "%DB_LABSTUDYDATA";

/// True when the current process may delete laboratory data.
ClassMethod CanDelete() As %Boolean
{
    quit $SYSTEM.Security.Check(..#DBRESOURCE, "WRITE")
}

/// Guard to be called at the beginning of a sensitive operation.
/// Returns $$$OK when allowed; otherwise records the attempt and
/// returns a deliberately vague error.
ClassMethod Require(operation As %String) As %Status
{
    if ..CanDelete() {
        quit $$$OK
    }

    // The full detail goes to the trail, not to the user.
    do ##class(LabStudy.AuditEntry).Record(
        "SECURITY", "", "DENIED", operation,
        $USERNAME_" / "_$ROLES, ..#DBRESOURCE)

    quit $$$ERROR($$$GeneralError, "You are not allowed to perform this operation")
}

/// Prints the current security context.
ClassMethod Context() As %Status
{
    write "User:      ", $USERNAME, !
    write "Roles:     ", $ROLES, !
    write "Namespace: ", $NAMESPACE, !
    write "Can write ", ..#DBRESOURCE, ": ", ..CanDelete(), !
    quit $$$OK
}

}
```

Acrescente a `src/LabStudy/Exam.cls`:

```objectscript
/// Clinical note, stored encrypted as Base64(iv)"|"Base64(ciphertext).
Property ClinicalNote As %String(MAXLEN = 1000);

/// Encrypts and stores the clinical note.
Method SetClinicalNote(text As %String, key As %String) As %Status
{
    if text = "" {
        set ..ClinicalNote = ""
        quit $$$OK
    }

    set iv = $SYSTEM.Encryption.GenCryptRand(16)
    set cipher = $SYSTEM.Encryption.AESCBCEncrypt(text, key, iv)

    set ..ClinicalNote = $SYSTEM.Encryption.Base64Encode(iv)
                         _"|"
                         _$SYSTEM.Encryption.Base64Encode(cipher)
    quit $$$OK
}

/// Decrypts the clinical note, if the caller may read the data.
Method GetClinicalNote(key As %String) As %String
{
    if ..ClinicalNote = "" {
        quit ""
    }

    if '$SYSTEM.Security.Check("%DB_LABSTUDYDATA", "READ") {
        quit "(access denied)"
    }

    set iv     = $SYSTEM.Encryption.Base64Decode($PIECE(..ClinicalNote, "|", 1))
    set cipher = $SYSTEM.Encryption.Base64Decode($PIECE(..ClinicalNote, "|", 2))

    quit $SYSTEM.Encryption.AESCBCDecrypt(cipher, key, iv)
}
```

Acrescente a `src/LabStudy/Patient.cls`:

```objectscript
/// Deletes a patient only when the caller has the required privilege.
ClassMethod DeleteSecurely(id As %String) As %Status
{
    set sc = ##class(LabStudy.Security).Require("DeletePatient:"_id)
    if $$$ISERR(sc) {
        quit sc
    }

    quit ..%DeleteId(id)
}
```

E em `src/LabStudy/App.cls`:

```objectscript
Parameter VERSION = "0.8";

/// Prints the security context and the application users.
ClassMethod SecurityReport() As %Status
{
    do ##class(LabStudy.Security).Context()

    write !, "Application users:", !
    set id = ""
    for {
        set id = $ORDER(^LabStudy.UserD(id))
        quit:id=""

        set user = ##class(LabStudy.User).%OpenId(id)
        continue:'$ISOBJECT(user)

        write "  ", user.Login, " (", user.FullName, ") role=", user.AppRole
        write $SELECT(user.Active: "", 1: " [INACTIVE]"), !
    }
    quit $$$OK
}
```

Execução esperada:

```
LABSTUDY>DO ##class(LabStudy.User).%KillExtent()

LABSTUDY>WRITE ##class(LabStudy.User).Create("recepcao02","Carla Nunes","senhaForte1","reception"), !
1

LABSTUDY>WRITE ##class(LabStudy.User).Create("tecnico01","Paulo Reis","outraSenha2","technician"), !
2

LABSTUDY>WRITE ##class(LabStudy.User).Authenticate("recepcao02","errada"), !
0
LABSTUDY>WRITE ##class(LabStudy.User).Authenticate("recepcao02","senhaForte1"), !
1
LABSTUDY>WRITE ##class(LabStudy.User).RoleOf("tecnico01"), !
technician

LABSTUDY>SET key = $SYSTEM.Encryption.GenCryptRand(32)
LABSTUDY>SET e = ##class(LabStudy.Exam).%OpenId(1)
LABSTUDY>DO e.SetClinicalNote("Result compatible with mild anaemia", key)
LABSTUDY>WRITE $$$ISOK(e.%Save()), !
1

LABSTUDY>WRITE e.ClinicalNote, !
Kd8Zx2Mn4Vb1Qp==|R7Yt3Jf0Ws6Ru2Hd9Km4Pn1QaLd8Xr4Kp9Mz2Vb7Nc1QaWs==

LABSTUDY>WRITE e.GetClinicalNote(key), !
Result compatible with mild anaemia

LABSTUDY>DO ##class(LabStudy.App).SecurityReport()
User:      _SYSTEM
Roles:     %All
Namespace: LABSTUDY
Can write %DB_LABSTUDYDATA: 1

Application users:
  recepcao02 (Carla Nunes) role=reception
  tecnico01 (Paulo Reis) role=technician
```

**Por que cada decisão:**

- **`LabStudy.User` é uma base de usuários *da aplicação*, separada dos usuários *do IRIS*.** Essa é uma decisão de arquitetura comum e vale entendê-la: os usuários do IRIS controlam quem acessa o banco; os da aplicação controlam quem faz o quê nas telas do laboratório. Os dois níveis coexistem, e o de aplicação nunca substitui o do IRIS — ele o complementa.
- **`Authenticate` recusa usuário inativo.** Desativar é sempre preferível a apagar: o histórico de auditoria continua fazendo sentido quando o usuário ainda existe.
- **`Security.Require` é o ponto único de guarda.** Toda operação sensível chama esse método. Se amanhã a política mudar — passar a exigir um recurso próprio em vez do recurso do banco —, muda-se um lugar.
- **A mensagem devolvida por `Require` é deliberadamente vaga**, enquanto o detalhe completo (usuário, papéis, recurso, operação) vai para a trilha. Isso põe em prática o que discutimos na seção 5.6. Repare que `Require` **registra a tentativa negada** — e essa entrada sobrevive porque não estamos dentro de uma transação que será desfeita.
- **`DeleteSecurely` verifica ANTES de apagar**, e devolve imediatamente em caso de negativa. Nenhum efeito colateral ocorre antes da verificação.
- **Note que `%DeleteId` continua existindo e continua funcionando sem verificação.** Isso é honesto e importante: a guarda protege o **caminho oficial**, não o dado em si. Proteção real do dado exige que o usuário não tenha privilégio de escrita no banco — e é aí que o modelo do IRIS entra. Verificação em código complementa; não substitui a configuração de segurança.
- **`ClinicalNote` é cifrado, `ResultValue` não.** Escolha deliberada: o valor numérico precisa ser consultado, indexado, comparado — cifrá-lo inutilizaria tudo isso. Já o texto livre da nota clínica é lido inteiro ou não é lido. **Cifrar campos que você precisa consultar é um erro de projeto**, e reconhecer isso é conhecimento sênior.
- **A chave não é guardada em lugar nenhum do projeto.** É uma lacuna consciente e anotada: em um sistema real, ela viria de um cofre externo. Fingir que resolvemos isso guardando a chave numa constante seria pior do que deixar a lacuna visível.

---

## 8. Quiz do capítulo

**Q1.** Na terminologia de segurança do IRIS, o que é um **privilégio**?

- A) Um conjunto de papéis.
- B) A combinação de um recurso com uma permissão (`READ`, `WRITE` ou `USE`).
- C) Um usuário com acesso total.
- D) Um serviço de entrada no sistema.

---

**Q2.** Qual é a cadeia correta do modelo de segurança?

- A) recurso → usuário → papel → privilégio
- B) usuário → papéis → privilégios → recursos
- C) papel → recurso → usuário → privilégio
- D) privilégio → papel → recurso → usuário

---

**Q3.** Qual chamada verifica, a partir do código, se o processo pode escrever num banco?

- A) `$ROLES("%DB_X")`
- B) `$SYSTEM.Security.Check("%DB_X", "WRITE")`
- C) `$SYSTEM.Security.Login("%DB_X")`
- D) `$USERNAME = "%All"`

---

**Q4.** Por que `NEW $ROLES` é obrigatório antes de elevar privilégios?

- A) Porque `AddRoles` não funciona sem ele.
- B) Porque garante que os papéis originais sejam restaurados ao sair do bloco, inclusive em caso de erro.
- C) Porque limpa os papéis do usuário permanentemente.
- D) Porque é exigido pelo compilador.

---

**Q5.** Você precisa guardar uma senha. Qual técnica usar?

- A) Criptografia AES, para poder recuperá-la se o usuário esquecer.
- B) Base64, para ficar ilegível.
- C) Hash com sal, sem nunca guardar a senha.
- D) Texto puro, com acesso restrito à tabela.

---

**Q6.** Qual é a diferença fundamental entre hash e criptografia?

- A) Nenhuma; são sinônimos.
- B) Hash é reversível; criptografia não.
- C) Criptografia é reversível com a chave; hash é um caminho só de ida.
- D) Hash é mais seguro em todos os casos.

---

**Q7.** O sal usado no hash de senhas precisa ser secreto?

- A) Sim, tanto quanto a senha.
- B) Não; ele fica guardado em claro, e sua função é impedir que senhas iguais gerem hashes iguais.
- C) Sim, e deve ser o mesmo para todos os usuários.
- D) Não, e por isso pode ser fixo no código.

---

**Q8.** O que Base64 oferece do ponto de vista de segurança?

- A) Criptografia forte.
- B) Hash irreversível.
- C) Nada: é apenas uma representação de bytes em texto, decodificável por qualquer um.
- D) Assinatura digital.

---

**Q9.** Qual função gera valores aleatórios adequados para uso criptográfico?

- A) `$RANDOM(1000)`
- B) `$SYSTEM.Encryption.GenCryptRand(16)`
- C) `$HOROLOG`
- D) `$INCREMENT(^Seed)`

---

**Q10.** Por que o IV do AES deve ser diferente a cada operação de cifragem?

- A) Porque ele é a chave.
- B) Porque, sendo fixo, o mesmo texto produziria sempre a mesma cifra, revelando que dois registros contêm o mesmo dado.
- C) Porque o IRIS exige.
- D) Porque ele precisa ser secreto.

---

**Q11.** Qual parâmetro de classe ativa a segurança em nível de linha?

- A) `Parameter ROWLEVELSECURITY = 1;`
- B) `Parameter DEFAULTCONCURRENCY = 1;`
- C) `Parameter SECURITY = "row";`
- D) `Parameter READERLIST = 1;`

---

**Q12.** Como se define quais papéis podem ler cada linha, com segurança em nível de linha?

- A) Por um índice único.
- B) Por um `ClassMethod %SecurityPolicy()` que devolve uma lista de papéis.
- C) Por uma palavra-chave de propriedade.
- D) Pelo Portal, linha a linha.

---

**Q13.** Qual é a defesa correta contra injeção de SQL?

- A) Filtrar aspas simples do texto digitado.
- B) Usar consultas com parâmetros (`?`) e passar os valores separadamente.
- C) Cifrar a consulta.
- D) Verificar o comprimento do texto.

---

**Q14.** Qual afirmação sobre criptografia do banco (*data at rest*) está correta?

- A) Protege um campo contra um usuário que tem acesso legítimo à tabela.
- B) É configurada pelo desenvolvedor no código da classe.
- C) Protege o arquivo do banco, do disco e do backup contra quem os obtiver, e é configurada pelo administrador.
- D) Substitui a necessidade de hash de senhas.

---

**Q15.** Uma verificação de privilégio falhou. Que mensagem apresentar ao usuário?

- A) O nome do recurso e do papel que faltam, para ele pedir ao administrador.
- B) Uma mensagem genérica, registrando o detalhe completo na trilha de auditoria.
- C) A lista completa de papéis do usuário.
- D) O comando SQL que foi bloqueado.

---

### Gabarito comentado

**Q1 — Resposta: B.**
- **B está certa:** privilégio é sempre recurso + permissão, como `%DB_LABSTUDYDATA:READ`.
- **A está errada:** conjunto de papéis não é um conceito do modelo; papel é que é conjunto de privilégios.
- **C está errada:** isso descreve um usuário com `%All`.
- **D está errada:** isso é um serviço.

**Q2 — Resposta: B.**
- **B está certa:** o usuário tem papéis, que contêm privilégios, que apontam para recursos.
- **A, C e D estão erradas:** invertem ou embaralham a cadeia.

**Q3 — Resposta: B.**
- **B está certa:** `$SYSTEM.Security.Check(recurso, permissão)` devolve 1 ou 0.
- **A está errada:** `$ROLES` é uma variável com a lista de papéis, não uma função de verificação.
- **C está errada:** `Login` autentica um usuário.
- **D está errada:** comparar o nome do usuário é frágil e não é o mecanismo previsto.

**Q4 — Resposta: B.**
- **B está certa:** o `NEW` garante restauração ao sair do bloco por qualquer caminho, inclusive por exceção.
- **A está errada:** `AddRoles` funciona sem ele — e é justamente esse o perigo.
- **C está errada:** ele não altera a configuração do usuário.
- **D está errada:** não é exigência de compilação.

**Q5 — Resposta: C.**
- **C está certa:** senha se guarda como hash com sal, nunca em forma recuperável.
- **A está errada:** cifra é reversível; quem obtiver a chave obtém todas as senhas. E "recuperar a senha esquecida" não é requisito legítimo: o correto é redefinir.
- **B está errada:** Base64 não protege nada.
- **D está errada:** erro grave e indefensável.

**Q6 — Resposta: C.**
- **C está certa:** criptografia é ida e volta com a chave; hash é só ida.
- **A está errada:** são conceitos distintos com finalidades distintas.
- **B está errada:** inverte as definições.
- **D está errada:** cada um serve para um propósito; nenhum é universalmente melhor.

**Q7 — Resposta: B.**
- **B está certa:** o sal é guardado em claro e serve para diversificar, impedindo comparação em massa e tabelas pré-calculadas.
- **A está errada:** ele precisa ser lido para conferir a senha.
- **C está errada:** sal igual para todos anula o benefício.
- **D está errada:** sal fixo no código também anula o benefício; ele deve ser diferente por usuário.

**Q8 — Resposta: C.**
- **C está certa:** Base64 é codificação, não proteção.
- **A, B e D estão erradas:** ele não cifra, não gera hash e não assina.

**Q9 — Resposta: B.**
- **B está certa:** `GenCryptRand` gera aleatoriedade de qualidade criptográfica.
- **A está errada:** `$RANDOM` é previsível e não serve para segurança.
- **C está errada:** data e hora são altamente previsíveis.
- **D está errada:** um contador é o oposto de aleatório.

**Q10 — Resposta: B.**
- **B está certa:** IV fixo faz o mesmo texto produzir sempre a mesma cifra, vazando a informação de que dois registros são iguais.
- **A está errada:** o IV não é a chave.
- **C está errada:** a razão é criptográfica, não uma exigência arbitrária.
- **D está errada:** o IV não é secreto; precisa apenas ser único.

**Q11 — Resposta: A.**
- **A está certa:** `ROWLEVELSECURITY = 1` liga o recurso.
- **B está errada:** `DEFAULTCONCURRENCY` trata de travamento ao abrir objetos.
- **C e D estão erradas:** esses parâmetros não existem.

**Q12 — Resposta: B.**
- **B está certa:** o `ClassMethod %SecurityPolicy()` devolve, com `$LISTBUILD`, os papéis que podem ler aquela linha; o resultado fica em `%READERLIST`.
- **A está errada:** índices não têm papel nisso.
- **C está errada:** não é uma palavra-chave de propriedade.
- **D está errada:** a política é calculada por código, na gravação.

**Q13 — Resposta: B.**
- **B está certa:** com parâmetros, o valor nunca é interpretado como parte do comando.
- **A está errada:** filtrar caracteres é frágil e historicamente sempre foi contornado.
- **C está errada:** cifrar a consulta não tem relação com o problema.
- **D está errada:** o comprimento não impede injeção.

**Q14 — Resposta: C.**
- **C está certa:** é transparente para a aplicação, configurada pelo administrador, e protege o arquivo contra quem o obtiver fisicamente.
- **A está errada:** para quem já tem acesso legítimo à tabela, o dado aparece decifrado normalmente.
- **B está errada:** não se configura no código.
- **D está errada:** senhas continuam exigindo hash.

**Q15 — Resposta: B.**
- **B está certa:** mensagem discreta para o usuário, detalhe completo na trilha, onde só quem investiga vê.
- **A está errada:** entrega ao atacante os nomes de recursos e papéis.
- **C está errada:** expõe a configuração de segurança.
- **D está errada:** revela a estrutura interna do sistema.

---

## 9. Resumo relâmpago

1. O modelo do IRIS: **usuário → papéis → privilégios → recursos**. Mais **serviços** (formas de entrar) e **aplicações** (áreas com papéis próprios).
2. **Privilégio = recurso + permissão.** As permissões são `READ` (R), `WRITE` (W) e `USE` (U).
3. Recursos de banco chamam-se `%DB_<NOMEDOBANCO>`. Recursos de aplicação são criados por você no Portal.
4. **`$USERNAME`** e **`$ROLES`** dão o contexto atual; `$ROLES` reflete papéis de login **mais** papéis adicionados.
5. **`$SYSTEM.Security.Check(recurso, permissão)`** devolve 1 ou 0. É a verificação central no código.
6. **`$SYSTEM.Security.Login(usuário, senha)`** troca o contexto de segurança do processo.
7. **Elevação temporária: `new $ROLES` + `$SYSTEM.Security.AddRoles(...)`.** O `NEW` restaura os papéis ao sair, mesmo em caso de erro.
8. `AddRoles` só concede papéis que o administrador autorizou; não é cheque em branco.
9. **Menor privilégio**: dê o mínimo necessário; eleve temporariamente quando preciso.
10. **Hash é só ida; criptografia é ida e volta.** Senha usa **hash**; dado a recuperar usa **criptografia**.
11. **`$SYSTEM.Encryption.SHAHash(256, texto)`** — use 256 bits ou mais. O resultado é binário: converta com `Base64Encode`.
12. **Sal** é aleatório, diferente por usuário, guardado em claro. Impede que senhas iguais gerem hashes iguais.
13. **`GenCryptRand(n)`** para aleatoriedade criptográfica. Nunca `$RANDOM`.
14. **`AESCBCEncrypt(texto, chave, iv)`** e **`AESCBCDecrypt`**. O **IV deve ser diferente a cada cifragem** e é guardado junto, em claro.
15. **Base64 não é segurança**: é representação de bytes em texto.
16. **O problema difícil é guardar a chave.** Chave ao lado do dado protege muito pouco.
17. **Criptografia do banco** (administrador, protege o arquivo) ≠ **criptografia da aplicação** (desenvolvedor, protege o campo). São complementares.
18. **Segurança em nível de linha**: `Parameter ROWLEVELSECURITY = 1` + `ClassMethod %SecurityPolicy()` devolvendo `$LISTBUILD` de papéis; o resultado vai para `%READERLIST`, calculado **na gravação**.
19. **Injeção de SQL**: nunca cole valores no texto da consulta; use parâmetros `?`.
20. Verifique privilégio **na entrada da operação**, antes de qualquer efeito colateral, e no método — não só na tela.
21. **Mensagem discreta ao usuário, detalhe completo na trilha de auditoria.**
22. Não cifre campos que você precisa consultar, indexar ou comparar.

---

## 10. Cartões de memorização

**Frente:** O que é um privilégio no IRIS?
**Verso:** A combinação de um **recurso** com uma **permissão**: `READ`, `WRITE` ou `USE`.

**Frente:** Qual a cadeia do modelo de segurança?
**Verso:** Usuário → papéis → privilégios → recursos.

**Frente:** Como se chama o recurso do banco `LABSTUDYDATA`?
**Verso:** `%DB_LABSTUDYDATA`.

**Frente:** Como verificar um privilégio no código?
**Verso:** `$SYSTEM.Security.Check("recurso", "READ")` — devolve 1 ou 0.

**Frente:** O que `$ROLES` contém?
**Verso:** A lista de papéis ativos **agora**, somando papéis de login e papéis adicionados durante a execução.

**Frente:** Qual o padrão correto de elevação temporária de privilégio?
**Verso:** `new $ROLES` seguido de `$SYSTEM.Security.AddRoles("papel")`. Ao sair do bloco, os papéis originais voltam.

**Frente:** Por que `NEW $ROLES` é indispensável?
**Verso:** Porque restaura os papéis mesmo se o método terminar por erro. Sem ele, o privilégio elevado vaza.

**Frente:** Diferença entre hash e criptografia.
**Verso:** Criptografia é reversível com a chave (ida e volta). Hash é só ida — não há como recuperar o original.

**Frente:** Como se guarda uma senha corretamente?
**Verso:** Hash com sal. A senha em si nunca é armazenada.

**Frente:** Para que serve o sal?
**Verso:** Para que senhas iguais gerem hashes diferentes, impedindo comparação em massa e tabelas pré-calculadas.

**Frente:** O sal é secreto?
**Verso:** Não. Fica guardado em claro, ao lado do hash, porque é preciso para conferir.

**Frente:** Qual função calcula hash no IRIS?
**Verso:** `$SYSTEM.Encryption.SHAHash(bits, texto)` — use 256 ou mais. O resultado é binário.

**Frente:** Para que serve `Base64Encode`?
**Verso:** Para representar bytes como texto e poder guardá-los num campo de string. **Não é segurança.**

**Frente:** Qual função gera aleatoriedade adequada para segurança?
**Verso:** `$SYSTEM.Encryption.GenCryptRand(n)`. Nunca `$RANDOM`.

**Frente:** Quais funções cifram e decifram com AES?
**Verso:** `$SYSTEM.Encryption.AESCBCEncrypt(texto, chave, iv)` e `AESCBCDecrypt(cifra, chave, iv)`.

**Frente:** O IV precisa ser secreto? E único?
**Verso:** Secreto não. Único sim: deve ser diferente a cada cifragem, e é guardado junto da cifra.

**Frente:** Qual é o problema realmente difícil da criptografia de aplicação?
**Verso:** Guardar a chave. Chave ao lado do dado cifrado protege muito pouco.

**Frente:** Diferença entre criptografia do banco e da aplicação.
**Verso:** A do banco (administrador) protege o arquivo roubado. A da aplicação (desenvolvedor) protege um campo de quem tem acesso legítimo à tabela.

**Frente:** Como ativar segurança em nível de linha?
**Verso:** `Parameter ROWLEVELSECURITY = 1` mais um `ClassMethod %SecurityPolicy()` devolvendo `$LISTBUILD` dos papéis leitores.

**Frente:** Quando a lista de leitores de uma linha é calculada?
**Verso:** Na **gravação**. Linhas antigas mantêm a lista antiga até serem regravadas.

**Frente:** Como se evita injeção de SQL?
**Verso:** Usando parâmetros (`?`) e passando os valores separadamente — nunca colando texto na consulta.

**Frente:** Que mensagem dar ao usuário quando falta privilégio?
**Verso:** Uma genérica. O detalhe completo vai para a trilha de auditoria.

**Frente:** Por que não cifrar um campo numérico de resultado?
**Verso:** Porque campos cifrados não podem ser consultados, indexados nem comparados. Cifre texto livre, não chaves de busca.

---

Fim do domínio **T2 — Basic Programming**. O próximo capítulo abre o **T3 — IRIS Features**, começando por 3.1 (meios de armazenamento: globais, tabelas e o modelo multimodelo).

Digite CONTINUAR para o próximo capítulo.
