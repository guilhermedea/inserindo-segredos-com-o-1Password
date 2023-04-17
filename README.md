# Guia básico de uso do 1Password para a inserção de segredos em código

Nesses tipos de ambientes é comum o uso de credenciais de acesso para permitir a criação e destruição de infraestrutura. Geralmente essas credenciais ficam em arquivos de texto, muitas vezes expostas e requerendo passos extras para manter a segurança dessas informações, como encriptação de arquivos ou a adição do arquivo ao GitIgnore, forçando cada dev no ciclo a ter sua própria versão do arquivo, o que pode levar a erros de configuração e credencial que podem causar atrasos na produção. A opção de uso de segredos permite que informações de acesso passem diretamente para o provedor, sem que eles sequer fiquem expostos em forma de texto simples durante o processo.

## Requerimentos
- Uma conta no 1Password
- 1Password CLI instalado na máquina e configurado para uso
- Credenciais de acesso salvas em um vault no 1Password

## Sintaxe
A sintaxe para referênciar itens em um vault do 1Password é:

`op://vault/item/[sessão/]/campo`

Exemplo: para referenciar o campo "access_key", que está dentro da sessão Credentials no item AWS-Prag, dentro do Vault Dev, a sintaxe fica:

`op://Dev/AWS-Prag/Credentials/access_key`

É possível também referenciar arquivos anexos nos itens usando a mesma sintaxe.

`op://vault/item/[sessão/]/nome-do-arquivo`

Exemplo: para referenciar uma chave SSH dentro de um vault chamado Dev:

`op://Dev/ssh/sshkey.pem`

**Se uma referência inclui espaços em branco, é preciso que a referência esteja entre aspas duplas ("):**

`"op://Dev/AWS Prag/Credentials/access_key"`

Você pode automaticamente gerar referências para todos os campos em um determinado item através do seguinte comando:

`op item get [nome do item] --format json`

Desta forma, você já recebe a referência pronta na linha "reference".

Outra forma de gerar uma referência é através da [extensão do 1Password para o Visual Studio Code](https://marketplace.visualstudio.com/items?itemName=1Password.op-vscode). Com a extensão instalada, você pode invocar o comando **1Password: Get from 1Password** diretamente da Paleta de Comando (Ctrl + Shift + P). Após configurar o vault a ser utilizado, basta invocar o comando, inserir o nome do item e selecionar qual campo você quer inserir, que a extensão insere diretamente no código a referência desejada. Além disto, a extensão reconhece automaticamente valores que podem ser considerados segredos no seu código e oferece a possibilidade de salvá-los no 1Password diretamente do Visual Studio Code.

## Inserindo segredos no código

Para provisionar uma aplicação com os segredos, é preciso mapear estes segredos para **variaveis de ambiente** (*environment variables*). É possível realizar isto através de duas formas: exportando diretamente na linha de comando ou utilizando arquivos .env.

### Exportando diretamente pela linha de comando

Este processo é feito em dois passos. Primeiramente é feito a exportação do segredo para uma variável de ambiente através de um comando.

| Bash, Zsh, sh | fish    | PowerShell     |
|---------------|---------|----------------|
|export NOME_VAR=op://vault/item/[sessão/]/campo | set -x NOME_VAR op://vault/item/[sessão/]/campo | $Env:NOME_VAR = "op://vault/item/[sessão/]/campo" |

Em seguida é utilizado um comando do 1Password em conjunto com o comando para iniciar a sua aplicação ou código.

`op run -- [código para iniciar seu app]`

Neste processo, o 1Password passará as informações para a aplicação através da variável de ambiente. Desta forma, os segredos nunca ficam expostos durante o processo.

### Utilizando arquivos de ambiente (.env)

Arquivos de ambiente permitem definir múltiplos segredos de uma vez só, através de uma sintaxe CHAVE=VALOR e separados em linhas diferentes.


> Exemplo de arquivo de ambiente .env </br>
>
> AWS_ACCESS_KEY_ID="op://development/aws/Access Keys/access_key_id" </br>
> AWS_SECRET_ACCESS_KEY="op://development/aws/Access Keys/secret_access_key" </br>
> NOME_DA_VARIAVEL="VALOR_DA_VARIAVEL" </br>


Para utilizar o arquivo de ambiente com o comando `op run`, especifique o caminho do arquivo através da flag `--env-file`.

`op run --env-file="./[arquivo.env]" -- [comando da aplicação]`

> Exemplo de comando
>
> `op run --env=file="./teste.env" -- terraform plan`

Lembrando que as variáveis de ambiente precisam ser escritas exatamente da forma requerida. Se a documentação utiliza "AWS_ACCESS_KEY_ID", então é preciso que ela seja definida com exatamente a mesma grafia.

Arquivos de ambiente segue diversas regras. As básicas e principais são:

- Variaveis são definidas em declarações "CHAVE=VALOR" e separadas por linha.
- Variáveis podem ocupar mais de uma linha desde que estejam entre aspas simples (') ou duplas (").
- Linhas vazias são puladas.
- Linhas que comecem com # são consideradas comentários. Comentários também pode ser acrescentados inline após a declaração CHAVE=VALOR.
- Aspas simples (') e duplas (") não saem no output final. Uma declaração CHAVE="VALOR" resultará no mesmo que a declaração CHAVE=VALOR.
- Um valor definido no arquivo pode ser referenciado novamente através da sintaxe `$NOME_VAR` ou `${NOME_VAR}`:

> Exemplo de referência interna </br>
> PRIMEIRA_VARIÁVEL=VALOR </br>
> SEGUNDA_VARIÁVEL=${PRIMEIRA_VARIAVEL} </br>

Outras regras podem ser encontradas na [documentação](https://developer.1password.com/docs/cli/secrets-environment-variables).

### Utilizando o mesmo arquivo .env para diferentes ambientes

É possível utilizar o mesmo arquivo .env para diferentes ambientes de produção com diferentes segredos. Para isso, é preciso que a estrutura dos vaults seja o mesmo em ambos os ambientes. 


> Exemplo </br>
>
> dev/mysql/password</br>
> prod/mysql/password</br>


Com as duas estruturas de acordo, no arquivo de ambiente, insira uma variável ($NOME_VARIÁVEL) no lugar do nome do vault. No exemplo a seguir, a variável se chama $APP_ENV.

> MYSQL_DATABASE = "op://$APP_ENV/mysql/database" </br>
> MYSQL_USERNAME = "op://$APP_ENV/mysql/username" </br>
> MYSQL_PASSWORD = "op://$APP_ENV/mysql/password" </br>

Em seguida passe o nome do ambiente como uma variável externa e em seguida execute o comando `op run` com o caminho do arquivo normalmente.

| Bash, Zsh, sh, fish    |
|------------------------|
| APP_ENV=dev op run --env-file="./app.env" -- [comando-do-app] | 

| PowerShell |
|------------|
| $ENV:APP_ENV = "dev" |
| op run --env-file="./app.env" -- [comando-do-app] |

No exemplo acima, ao passar o valor "dev" como uma variável externa, o arquivo de ambiente usará as informações no vault "Dev". Se passar o valor externo como "Prod" o arquivo usará as informações do vault "Prod."

## Notas finais
Este tutorial tem como propósito o ensino básico do uso de segredos através do 1Password. Utilizando desta forma, os segredos nunca ficam expostos em arquivos de texto, e os arquivos de ambiente podem tranquilamente ser incluídos no sistema de versionamento utilizado, já que os segredos só são passados na execução mediante a autorização pelo 1Password.

Para provisionar segredos em ambientes de produção existe o serviço 1Password Connect Server, que pode ser utilizado em ambientes de CI. 






