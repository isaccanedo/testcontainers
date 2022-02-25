# Testando scripts de migração de banco de dados com Spring Boot e Testcontainers

### A migração de banco de dados com ferramentas como Flyway ou Liquibase 
Para a criação de scripts SQL e a execução deles em um banco de dados. Embora o banco de dados seja uma dependência externa, temos que testar os scripts SQL, pois é o nosso código. Mas esse código não roda na aplicação que desenvolvemos e não pode ser testado com testes unitários.

### Principais conclusões
- Usar um banco de dados in-memory para testes de integração causará problemas de compatibilidade em nossos scripts SQL entre o banco de dados in-memory e o banco de dados de produção;
- Usando Testcontainers, podemos facilmente criar um container Docker com o banco de dados de produção para nossos testes.

### Prática comum
Existe uma abordagem muito comum e conveniente para testar scripts de migração de banco de dados com Flyway em tempo de compilação.

É uma combinação de suporte à migração Flyway no Spring Boot e um banco de dados na memória como H2. Nesse caso, a migração do banco de dados começa sempre que o contexto do aplicativo Spring é iniciado e os scripts SQL são executados em um banco de dados H2 com Flyway.

É fácil e rápido. Mas é bom?

### O problema de usar um banco de dados na memória para testes
O H2 geralmente não é o banco de dados que usamos na produção ou em outros ambientes semelhantes à produção. Quando testamos os scripts SQL com o banco de dados H2, não temos ideia de como a migração seria executada no ambiente de produção.

### Banco de dados na memória em produção
Se usarmos um banco de dados na memória em produção, essa abordagem será adequada. Podemos apenas testar o aplicativo com um banco de dados integrado como o H2. Nesse caso, esses testes são completamente válidos e significativos.

O H2 tem modos de compatibilidade para disfarçar como outros bancos de dados. Isso pode incluir nosso banco de dados de produção. Com esses modos, podemos iniciar o banco de dados H2 e ele se comportará, por exemplo, como um banco de dados PostgreSQL.

Mas ainda há diferenças. O código SQL para um H2 ainda pode parecer diferente do código para PostgresSQL.

Vejamos este script SQL:

```
CREATE TABLE car
(
  id  uuid PRIMARY KEY,
  registration_number VARCHAR(255),
  name  varchar(64) NOT NULL,
  color varchar(32) NOT NULL,
  registration_timestamp INTEGER
);
```

Este script pode ser executado em um banco de dados H2 e também em um banco de dados PostgreSQL.

Agora queremos alterar o tipo da coluna registration_timestamp de INTEGER para timestamp com fuso horário e claro, queremos migrar os dados desta coluna. Então, escrevemos um script SQL para migrar a coluna registration_timestamp:

```
ALTER TABLE car
  ALTER COLUMN registration_timestamp SET DATA TYPE timestamp with time zone
   USING
   timestamp with time zone 'epoch' +
    registration_timestamp * interval '1 second';
```

Este script não funcionará para H2 com modo PostgreSQL, porque a cláusula USING não funciona com ALTER TABLE para H2.

Dependendo do banco de dados que temos em produção, podemos ter recursos específicos do banco de dados nos scripts SQL. Outro exemplo seria usar herança de tabelas no PostgreSQL com a palavra-chave INHERITS, que não é suportada em outros bancos de dados.

Poderíamos, é claro, manter dois conjuntos de scripts SQL, um para H2, para ser usado nos testes, e outro para PostgreSQL, para ser usado em produção:

Mas agora,:

- temos que configurar perfis Spring Boot para diferentes pastas com scripts,
- temos que manter dois conjuntos de scripts,
- e o mais importante, não podemos testar scripts da pasta postgresql em tempo de compilação.

Se quisermos escrever um novo script com alguns recursos que não são suportados pelo H2, temos que escrever dois scripts, um para H2 e outro para PostgreSQL. Além disso, temos que encontrar uma maneira de obter os mesmos resultados com os dois scripts.

Se testarmos os scripts do banco de dados com o banco de dados H2 e nosso teste estiver verde, não saberemos nada sobre o script V1_2__change_column_type.sql da pasta postgresql.

Esses testes nos dariam uma falsa sensação de segurança!

### Usando um ambiente de produção para testar scripts de banco de dados
Existe outra abordagem para testar a migração de banco de dados: podemos testar a migração de banco de dados com um banco de dados H2 em tempo de compilação e, em seguida, implantar nosso aplicativo em um ambiente semelhante à produção e deixar os scripts de migração serem executados nesse ambiente com o banco de dados semelhante à produção, por exemplo , PostgreSQL.

Essa abordagem nos alertará se algum script não estiver funcionando com o banco de dados de produção, mas ainda tem desvantagens:

```
- Bugs são descobertos tarde demais,
- é difícil encontrar erros,
- e ainda temos que manter dois conjuntos de scripts SQL.
```

Vamos imaginar que testamos a migração com o banco de dados H2 durante o tempo de construção do aplicativo e os testes são verdes. A próxima etapa é entregar e implantar o aplicativo em um ambiente de teste. Leva tempo. Se a migração no ambiente de teste falhar, seremos notificados tarde demais, talvez vários minutos depois. Isso retarda o ciclo de desenvolvimento.

Além disso, essa situação é muito confusa para os desenvolvedores, porque não podemos depurar erros como em nosso teste de unidade. Nosso teste de unidade com H2 estava verde, afinal, e o erro só aconteceu no ambiente de teste.

### Usando contêineres de teste
Com Testcontainers, podemos testar a migração do banco de dados em um contêiner Docker do banco de dados de produção do nosso código. Na máquina do desenvolvedor ou no servidor CI.

Testcontainers é uma biblioteca Java que facilita a inicialização de um contêiner do Docker a partir de nossos testes.

Claro, teremos que instalar o Docker para executá-lo. Depois disso, podemos criar algum código de inicialização para teste:

```
@ContextConfiguration(
  initializers = AbstractIntegrationTest.Initializer.class)
public class AbstractIntegrationTest {

  static class Initializer implements 
       ApplicationContextInitializer<ConfigurableApplicationContext> {

    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>();

    private static void startContainers() {
      Startables.deepStart(Stream.of(postgres)).join();
      // we can add further containers 
      // here like rabbitmq or other databases
    }

    private static Map<String, String> createConnectionConfiguration() {
      return Map.of(
          "spring.datasource.url", postgres.getJdbcUrl(),
          "spring.datasource.username", postgres.getUsername(),
          "spring.datasource.password", postgres.getPassword()
      );
    }


    @Override
    public void initialize(
        ConfigurableApplicationContext applicationContext) {
      
      startContainers();

      ConfigurableEnvironment environment = 
        applicationContext.getEnvironment();

      MapPropertySource testcontainers = new MapPropertySource(
          "testcontainers",
          (Map) createConnectionConfiguration()
      );

      environment.getPropertySources().addFirst(testcontainers);
    }
  }
} 
```

AbstractIntegrationTest é uma classe abstrata que define um banco de dados PostgreSQL e configura a conexão com este banco de dados. Outras classes de teste que precisam de acesso ao banco de dados PostgreSQL podem estender essa classe.

Na anotação @ContextConfiguration, adicionamos um ApplicationContextInitializer que pode modificar o contexto do aplicativo quando ele é inicializado. O Spring chamará o método initialize().

Dentro de initialize(), primeiro iniciamos o container Docker com um banco de dados PostgreSQL. O método deepStart() inicia todos os itens do Stream em paralelo. Poderíamos adicionar contêineres Docker, por exemplo, RabbitMQ, Keycloak ou outro banco de dados. Para simplificar, estamos iniciando apenas um contêiner Docker com o banco de dados PostgreSQL.

Em seguida, chamamos createConnectionConfiguration() para criar um mapa das propriedades de conexão do banco de dados. A URL para o banco de dados, nome de usuário e senha são criados pelos Testcontainers automaticamente. Assim, nós os obtemos da instância testcontainers postgres e os retornamos.

Também é possível definir esses parâmetros manualmente no código, mas é melhor deixar Testcontainers gerá-los. Quando deixamos Testcontainers gerar o jdbcUrl, ele inclui a porta da conexão do banco de dados. A porta aleatória fornece estabilidade e evita possíveis conflitos na máquina de outro desenvolvedor ou servidor de compilação.

Por fim, adicionamos essas propriedades de conexão de banco de dados ao contexto Spring criando um MapPropertySource e adicionando-o ao ambiente Spring. O método addFirst() adiciona as propriedades aos contextos com maior precedência.

Agora, se quisermos testar scripts de migração de banco de dados, temos que estender a classe e criar um teste de unidade.

```
@SpringBootTest
class TestcontainersApplicationTests extends AbstractIntegrationTest {

  @Test
  void migrate() {
  // migration starts automatically,
  // since Spring Boot runs the Flyway scripts on startup
  }

}
```

A classe AbstractIntegrationTest pode ser usada não apenas para testar scripts de migração de banco de dados, mas também para quaisquer outros testes que precisem de uma conexão com o banco de dados.

Agora podemos testar a migração de scripts SQL com Flyway usando um banco de dados PostgreSQL em tempo de compilação.

Temos todas as dependências em nosso código e podemos criar um ambiente de teste próximo à produção em qualquer lugar.

### Desvantagens
Como mencionamos acima, temos que instalar o Docker em todas as máquinas em que queremos construir o aplicativo. Pode ser um laptop de desenvolvedor ou um servidor de compilação de CI.

Além disso, os testes que interagem com Testcontainers são mais lentos do que o mesmo teste com um banco de dados na memória, porque o contêiner do Docker precisa ser ativado.

# Conclusão
Testcontainers oferece suporte ao teste do aplicativo com testes de unidade usando contêineres do Docker com o mínimo de esforço.

Os testes de migração de banco de dados com Testcontainers fornecem um comportamento de banco de dados semelhante à produção e melhoram significativamente a qualidade dos testes.

Não há necessidade de usar um banco de dados na memória para testes.
