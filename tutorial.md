# Tutorial: compilar, executar e acessar o PostgreSQL

Este tutorial descreve o ambiente já configurado nesta máquina Windows. Todos os comandos abaixo são para o **Prompt de Comando (CMD)**, salvo quando indicado.

## 1. Ambiente do projeto

| Componente | Configuração |
| --- | --- |
| Pasta do projeto | `C:\Users\Felipe\Desktop\Haskell\yesod-minimal-psql-static` |
| Stack configurado | `3.11.1` |
| Snapshot em `stack.yaml` | `lts-24.67` |
| GHC usado pelo projeto | `9.10.3` |
| Executável | `ProjetoWeb` |
| Endereço da aplicação | `http://localhost:8080` |
| PostgreSQL | `localhost:5432` |
| Banco de dados | `proj` |
| Usuário do banco | `fulano` |

O Stack seleciona o compilador do projeto. O comando `ghc --version` executado diretamente pode mostrar outra versão instalada globalmente.

## 2. Compilar o projeto (build)

Abra o CMD e entre na pasta que contém `stack.yaml`:

```cmd
cd /d "C:\Users\Felipe\Desktop\Haskell\yesod-minimal-psql-static"
```

Confira o Stack e o GHC selecionado para este projeto:

```cmd
stack --numeric-version
stack exec ghc -- --numeric-version
```

Os resultados esperados neste ambiente são `3.11.1` e `9.10.3`, respectivamente.

Compile:

```cmd
stack build
```

O primeiro build pode demorar porque o Stack baixa e compila as dependências. Os builds seguintes aproveitam o cache. Não é necessário iniciar o servidor PostgreSQL para compilar neste ambiente, onde as bibliotecas nativas já estão instaladas.

Depois de alterar o código Haskell, execute `stack build` novamente antes de iniciar o executável. `stack exec ProjetoWeb` executa o binário existente; ele não recompila automaticamente suas alterações.

## 3. Iniciar o PostgreSQL

O banco já foi criado nesta máquina, mas o servidor não foi registrado como serviço do Windows e não inicia automaticamente com o sistema.

Na pasta do projeto, execute:

```cmd
powershell -NoProfile -ExecutionPolicy Bypass -File .\.stack-work\tools\start-postgres.ps1
```

O script verifica se esta instância está ativa e a inicia quando necessário. A opção `ExecutionPolicy Bypass` vale apenas para esse processo do PowerShell; ela não altera permanentemente a política do Windows.

O script fica em `.stack-work`, uma pasta ignorada pelo Git. Ele existe neste ambiente, mas não acompanha um novo clone do repositório. A seção 5 também mostra como iniciar o banco diretamente, sem depender desse script.

## 4. Executar o projeto

Com o build concluído e o PostgreSQL ativo, execute na pasta do projeto:

```cmd
stack exec ProjetoWeb
```

Abra no navegador:

```text
http://localhost:8080
```

Deixe esse terminal aberto enquanto utiliza a aplicação. Para encerrar o servidor web, pressione **Ctrl+C**. Isso não encerra o PostgreSQL, que é um processo separado.

A conexão com o banco está definida em `connStr`, no arquivo `app/Main.hs`. Na inicialização, o projeto executa automaticamente `runMigration migrateAll`, aplicando as migrações necessárias ao banco configurado.

## 5. Acessar o PostgreSQL pelo CMD, separado do projeto

O `psql` é o cliente de terminal do PostgreSQL. Você pode usá-lo em outro CMD, a partir de qualquer pasta, sem iniciar a aplicação Haskell. Apenas o servidor PostgreSQL precisa estar ativo.

### Onde estão os executáveis e os dados

Os executáveis foram instalados no ambiente MSYS2 gerenciado pelo Stack:

```text
%LOCALAPPDATA%\Programs\stack\x86_64-windows\msys2-20240727\clang64\bin
```

Os dados desta instância estão separados, em:

```text
%LOCALAPPDATA%\ProjetoWeb\postgresql\data
```

O arquivo de log está em:

```text
%LOCALAPPDATA%\ProjetoWeb\postgresql\postgresql.log
```

Não foi feita uma instalação tradicional com serviço do Windows, nem adicionada a pasta do PostgreSQL ao PATH permanente. Por isso, os comandos abaixo usam caminhos completos. Esses caminhos correspondem à configuração desta máquina.

### Verificar se o servidor está disponível

```cmd
"%LOCALAPPDATA%\Programs\stack\x86_64-windows\msys2-20240727\clang64\bin\pg_isready.exe" -h localhost -p 5432
```

Com o servidor ativo, a resposta contém `accepting connections`.

Se estiver parado, inicie a instância já criada:

```cmd
"%LOCALAPPDATA%\Programs\stack\x86_64-windows\msys2-20240727\clang64\bin\pg_ctl.exe" -D "%LOCALAPPDATA%\ProjetoWeb\postgresql\data" -l "%LOCALAPPDATA%\ProjetoWeb\postgresql\postgresql.log" -w start
```

Esse comando funciona independentemente da pasta atual. Não é necessário inicializar o banco novamente.

### Conectar ao banco

```cmd
"%LOCALAPPDATA%\Programs\stack\x86_64-windows\msys2-20240727\clang64\bin\psql.exe" -h localhost -p 5432 -U fulano -d proj -W
```

Quando solicitado, informe a senha definida em `connStr` no arquivo `app/Main.hs`. A senha não aparece na tela enquanto é digitada. Ela não foi copiada para este tutorial.

Após conectar, você verá um prompt semelhante a `proj=>`.

### Comandos úteis dentro do psql

| Comando | Função |
| --- | --- |
| `\conninfo` | Mostrar a conexão atual |
| `\l` | Listar os bancos |
| `\dt` | Listar as tabelas |
| `\d pessoa` | Mostrar a estrutura da tabela `pessoa` |
| `SELECT * FROM pessoa;` | Consultar pessoas |
| `SELECT * FROM blog_post;` | Consultar publicações |
| `\q` | Sair do psql e voltar ao CMD |

Comandos SQL terminam com ponto e vírgula (`;`). Os comandos do psql iniciados com barra invertida, como `\dt` e `\q`, não precisam dele.

### Usar apenas o comando psql nesta janela

Opcionalmente, adicione a pasta ao PATH da janela atual do CMD:

```cmd
set "PATH=%LOCALAPPDATA%\Programs\stack\x86_64-windows\msys2-20240727\clang64\bin;%PATH%"
psql -h localhost -p 5432 -U fulano -d proj -W
```

Essa alteração termina quando a janela é fechada. Ela não modifica o PATH permanente do Windows nem cria um serviço.

### Encerrar o PostgreSQL

Encerre primeiro a aplicação com Ctrl+C e saia das sessões do psql. Depois, no CMD:

```cmd
"%LOCALAPPDATA%\Programs\stack\x86_64-windows\msys2-20240727\clang64\bin\pg_ctl.exe" -D "%LOCALAPPDATA%\ProjetoWeb\postgresql\data" -m fast -w stop
```

Esse comando encerra o servidor, desconecta clientes restantes e desfaz transações ainda abertas. Os dados já confirmados permanecem armazenados.

## 6. Solução de problemas

### `Executable named ProjetoWeb not found` ou `implicit global project`

Entre na pasta correta e compile antes de executar:

```cmd
cd /d "C:\Users\Felipe\Desktop\Haskell\yesod-minimal-psql-static"
stack build
stack exec ProjetoWeb
```

### `Connection refused` em `localhost:5432`

Verifique o servidor com `pg_isready` e inicie-o conforme a seção 3 ou 5. Esse erro significa que a conexão ao servidor foi recusada; ele ocorre antes da validação de usuário e senha.

### Falha de autenticação

Confira o usuário, a senha e o banco em `app/Main.hs`. Alterar a senha nesse arquivo não altera a senha do usuário dentro do PostgreSQL. As duas configurações precisam corresponder.

### `psql` não é reconhecido pelo CMD

Use o caminho completo da seção 5 ou adicione a pasta ao PATH da janela atual.

### Falha de download envolvendo `SSLKEYLOGFILE` ou `aswMonFltProxy`

Foi identificado neste ambiente um problema com o valor herdado de `SSLKEYLOGFILE`. Se esse erro reaparecer durante downloads do Stack, limpe a variável apenas no CMD atual e tente novamente:

```cmd
set "SSLKEYLOGFILE="
stack build
```

Isso não desativa a verificação TLS nem altera permanentemente as variáveis do Windows.
