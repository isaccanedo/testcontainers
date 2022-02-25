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

O problema de usar um banco de dados na memória para testes
O H2 geralmente não é o banco de dados que usamos na produção ou em outros ambientes semelhantes à produção. Quando testamos os scripts SQL com o banco de dados H2, não temos ideia de como a migração seria executada no ambiente de produção.