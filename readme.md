# Prisma Migrations

- Em teoria, o uso de migrations deve ser simples, ao declarar um model no seu `schema.prisma` e rodar o comando, nossa migration será executada pela CLI do prisma e nossos models serão convertidos para tabelas, o mesmo vai para colunas e relacionamentos estabelecidos entre as entidades

```shell
    prisma migrate dev --name <migration_name>
```

- Concomitante a isso esse comando irá gerar um arquivo `.sql` contendo o nome da sua migração e o respectivo sql que foi necessário para sincronizar as alterações do `schema.prisma` com o banco de dados.

## Como adicionar o prisma a um projeto existente

Nesse tópico, a documentação recomenda alguns passos específicos

- Fazer uma introspecção do seu banco atual para atualizar o seu schema.prisma -> `prisma db pull`

- Criar uma migration baseline: Esse processo somente é necessário para uma database que já existia previamente antes da implementação do Prisma || Database contém informações que precisam ser mantidas ( como em produção )

- Baselines dizem a engine do prisma para assumir que uma ou mais migração já foram aplicadas. Isso serve para prevenir as migrações de falharem ao por tentarem criar colunas ou tabelas já existentes no banco de dados.

- Caso já exista um diretório prisma/migrations, devemos mover, renomear ou arquivar esse diretório

- Podemos gerar uma migration e salvar em um arquivo usando o comando

```shell
npx prisma migrate diff \
--from-empty\
--to-schema-datamodel prisma/schema.prisma \
--script > prisma/migrations/0_init/migration.sql
```

- Caso necessário, garanta que seu arquivo de migração `.sql` contém o SQL necessário para contornar recursos de banco de dados que ainda não tem suporte no Prisma. Para uma lista desses recursos sem suporte, acesse: https://www.prisma.io/docs/orm/prisma-migrate/workflows/unsupported-database-features

# Development x Produção

## Development

Em um ambiente de desenvolvimento, usamos o comando `npx migrate dev`, de acordo com a documentação, esse comando funciona da seguinte forma:

- Executa novamente o histórico de migrações em um recurso que a documentação denomina de shadow database (um database temporário que é criado e deletado automaticamente) com o intuito de identificar problemas como inconsistências entre as migrações e o estado da database, tal como perca de dados que podem ser consequência da migração. Para conhecer mais esse recurso, acesse: https://www.prisma.io/docs/orm/prisma-migrate/understanding-prisma-migrate/shadow-database

- Aplica as migrações pendentes na shadow database

- Detecta mudanças no seu `schema.prisma` e gera novos arquivos de migração que contemplem tais alterações

- Aplica todas as migrações na database de development e realiza um update em \_prisma_migrations para documentar as alterações

- Engatilha a a geração dos artefatos como, por exemplo, o Prisma Client

Esse comando irá solicitar que você resete a sua database nos seguintes cenários

- **Conflitos no histórico de migração causados por migrações modificadas manualmente ou ausentes**

- **No final do processo, o estado final do `schema.prisma` difere do histórico de migrações**

Em linhas gerais a shadow database identifica incompatibilidades da seguinte forma:

- Realiza uma introspecção da shadow database e gera um schema.prisma que seria o ideal após todas as migrações serem executadas.

- Compara esse schema.prisma ideal com o estado atual do database de desenvolvimento.

- Informa caso existam divergências sobre o estado dos schemas.

## Production

Em produção ou ambientes de teste, devemos rodar o comando

```shell
npx prisma migrate deploy
```

É altamente recomendado que esse comando não seja executado de forma manual e faça parte de uma pipeline automatizada de CI/CD como demonstrado abaixo:

```yaml
name: Deploy
on:
  push:
    branches:
      - main
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repo
        uses: actions/checkout@v3
      - name: Setup Node
        uses: actions/setup-node@v3
      - name: Install dependencies
        run: npm install
      - run: npm run build
      - name: Apply all pending migrations to the database
        run: npx prisma migrate deploy
        env:
          DATABASE_URL: ${{ secrets.DATABASE_URL }}
```

Esse comando funciona da seguinte forma:

- Compara as migrações aplicadas com o histórico de migrações e te avisa caso alguma migração tenha sido alterada.

- Esse comando não utiliza o recurso de shadow databases

- Aplica as alterações

Esse comando não irá te alertar nos seguintes casos:

- Se alguma migração já executada está ausente do histórico de migrações.

- Não detecta incompatibilidades entre schemas

- Não reseta a databse ou gera artefatos

## Database Lock

A engine do prisma irá locar a database na execução dos seguintes comandos

```shell
prisma migrate deploy
prisma migrate dev
prisma migrate resolve

```

Essa medida de precaução é tem como objetivo garantir que diversos comandos não rodem de forma paralela ( caso de duas PR's em processo de deploy após serem aprovadas) esse lock é de 10 segundos e não é configurável

- Desde a versão `5.3.0` esse lock pode ser desabilitado por meio da variável de ambiente `PRISMA_SCHEMA_DISABLE_ADVISORY_LOCK`

## Histórico de migrações

Esse histórico é a linha do tempo das alterações realizadas em seu banco de dados, e é composto por um diretório no projeto `prisma/migrations` com um arquivo de `migration.sql` para cada migração executada, acompanhada de um timestamp de quando foi criada. Tal como uma tabela diretamente no banco denominada `_prisma_migrations` que usada para validar se uma migration foi executada, alterada ou deletada

Esse diretório `migrations` é a fonte de verdade sobre as alterações no seu banco de dados.

Em linhas gerais não devemos deletar ou modificar migrations que já foram executadas, isso pode fazer com que apareçam inconsistências entre desenvolvimento e produção. Na qual podem gerar bugs como `prisma migrate deploy` sempre emitindo um aviso de que o existem arquivos de migração modificados ou ausentes. Ou em um pior caso bugs que acontecem em produção e não são replicados em ambiente de desenvolvimento.

Nesse caso, a ação mais recomendada seria restaurar o arquivo de migração alterado para o seu estado original.
