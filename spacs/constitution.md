# Constitution Estacionamento Rotativo 

Regras persistentes. Valem para todo código gerado. Em conflito, este arquivo prevalece sobre `spec.md`, `plan.md`, `tests.md` e `tasks.md`.


## 1. Escopo

- Objeto: somente a API REST de bilhetes de estacionamento rotativo (abrir, encerrar, cancelar, listar ativos, histórico por placa, relatório diário).
- Fora de escopo: back-office, front-end, autenticação, pagamentos, persistência durável.



## 2. Parâmetros da Variante 

| Parâmetro              | Valor        | Propriedade Spring                     |
|------------------------|--------------|----------------------------------------|
| TARIFA_HORA_CENTAVOS   | `<preencher>` | `estacionamento.tarifa-hora-centavos`  |
| FRACAO_MINUTOS         | `<preencher>` | `estacionamento.fracao-minutos`        |
| TETO_DIARIO_CENTAVOS   | `<preencher>` | `estacionamento.teto-diario-centavos`  |
| TOLERANCIA_MINUTOS     | `<preencher>` | `estacionamento.tolerancia-minutos`    |
| PORTA_SERVICO          | `<preencher>` | `server.port`                          |

- Os parâmetros são lidos por uma classe `@ConfigurationProperties(prefix = "estacionamento")` validada na inicialização (`@Validated`, todos `> 0`, exceto tolerância que aceita `0`, `10` ou `15`).
- A aplicação DEVE falhar ao subir se algum parâmetro estiver ausente ou inválido.
- `60 % FRACAO_MINUTOS` DEVE ser `0` (validado na inicialização), garantindo divisão exata da tarifa.

## 3. Stack Tecnológica

| Item               | Escolha                                                        |
|--------------------|----------------------------------------------------------------|
| Linguagem          | Java 21                                                  |
| Framework          | Spring Boot 3.x                                                |
| Módulos            | `spring-boot-starter-web`, `spring-boot-starter-validation`, `spring-boot-starter-data-jpa` |
| Banco de dados     | H2 em memória (`jdbc:h2:mem:estacionamento`), schema gerado pelo JPA |
| Build              | Maven (wrapper `./mvnw` versionado)                            |
| Serialização JSON  | Jackson (padrão do Spring Boot)                                |
| Testes             | JUnit 5, Spring Boot Test, MockMvc, AssertJ                    |
| Tempo              | `java.time` exclusivamente (proibido `Date`, `Calendar`)       |

Proibido adicionar dependências fora desta lista sem atualizar esta constituição.


## 4. Arquitetura em Camadas

com.estacionamento
├── controller   → recebe HTTP, converte DTO, delega ao service. SEM regra de negócio.
├── service      → regras de negócio, cálculo de valor, transições de status, transações.
├── repository   → interfaces Spring Data JPA. SEM regra de negócio.
├── domain       → entidade JPA `Bilhete` e enum `StatusBilhete`.
├── dto          → records de request/response (contrato HTTP).
├── exception    → exceções de domínio + `GlobalExceptionHandler` (@RestControllerAdvice).
└── config       → `@ConfigurationProperties`, `Clock`, configuração do Jackson.