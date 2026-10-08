# Plan — API de Estacionamento Rotativo (Bilhetes)

> Plano técnico de implementação. Traduz o `spec.md` em decisões de código, obedecendo ao `constitution.md`.
> Em conflito, prevalece: contrato do enunciado > `constitution.md` > `spec.md` > `plan.md`.
> Detalhamento de casos de teste fica em `tests.md`; lista executável de tarefas fica em `tasks.md`.

---

## 1. Visão Geral da Solução

- Aplicação Spring Boot única, empacotada como JAR executável, escutando em `PORTA_SERVICO`.
- Arquitetura em camadas: `controller → service → repository → domain`.
- Persistência em H2 em memória via Spring Data JPA.
- Todo cálculo de tempo usa `Clock` injetado; todo cálculo monetário usa `long`.
- Erros tratados centralmente por `GlobalExceptionHandler`, sempre no formato `{"erro": "<codigo>"}`.

---

## 2. Estrutura do Projeto

Pacote base: `com.estacionamento`.

| Pacote / Pasta | Arquivos |
|----------------|----------|
| raiz do projeto | `pom.xml`, `mvnw`, `mvnw.cmd`, `.mvn/` |
| `com.estacionamento` | `EstacionamentoApplication` |
| `config` | `EstacionamentoProperties`, `ClockConfig` |
| `controller` | `BilheteController`, `RelatorioController` |
| `service` | `BilheteService`, `RelatorioService`, `CalculadoraTarifa`, `ValidadorEntrada`, `BilheteMapper` |
| `repository` | `BilheteRepository` |
| `domain` | `Bilhete`, `StatusBilhete` |
| `dto` | `AbrirBilheteRequest`, `BilheteResponse`, `EncerramentoResponse`, `RelatorioDiarioResponse`, `ErroResponse` |
| `exception` | `ApiException`, `PlacaInvalidaException`, `EntradaInvalidaException`, `DataInvalidaException`, `BilheteNaoEncontradoException`, `BilheteJaEncerradoException`, `BilheteNaoAbertoException`, `BilheteEmAbertoException`, `GlobalExceptionHandler` |
| `src/main/resources` | `application.properties` |
| `src/test/.../service` | `CalculadoraTarifaTest`, `ValidadorEntradaTest` |
| `src/test/.../config` | `EstacionamentoPropertiesTest` |
| `src/test/.../controller` | `BilheteControllerIT`, `RelatorioControllerIT`, `ErrosGeraisIT` |
| `src/test/.../support` | `RelogioAjustavel`, `RelogioTestConfig` |

---

## 3. Dependências (`pom.xml`)

| Dependência | Escopo | Uso |
|-------------|--------|-----|
| `spring-boot-starter-parent` 3.x | parent | `java.version` = 21 |
| `spring-boot-starter-web` | compile | API REST |
| `spring-boot-starter-validation` | compile | validação das properties |
| `spring-boot-starter-data-jpa` | compile | repositório |
| `com.h2database:h2` | runtime | banco em memória |
| `spring-boot-starter-test` | test | JUnit 5, MockMvc, AssertJ |
| `spring-boot-configuration-processor` | optional | metadados das properties |

Nenhuma outra dependência (Lombok, MapStruct etc. proibidos pela constituição).

---

## 4. Configuração

### 4.1 `application.properties`
Conforme seção 10 da constituição: porta, parâmetros da variante, H2 em memória, `ddl-auto=create-drop`, `open-in-view=false`, `include-stacktrace=never`.

### 4.2 `EstacionamentoProperties`
Record anotado com `@Validated` e `@ConfigurationProperties(prefix = "estacionamento")`, registrado via `@ConfigurationPropertiesScan` na classe principal.

| Campo | Tipo | Validação |
|-------|------|-----------|
| `tarifaHoraCentavos` | `long` | `@Positive` |
| `fracaoMinutos` | `int` | `@Positive` + `@AssertTrue` em `isFracaoDivisorDe60()` (`60 % fracao == 0`) |
| `tetoDiarioCentavos` | `long` | `@Positive` |
| `toleranciaMinutos` | `int` | `@PositiveOrZero` + `@AssertTrue` em `isToleranciaValida()` (0, 10 ou 15) |

- Falha de validação impede a aplicação de subir.

### 4.3 `ClockConfig`
- Bean `Clock` = `Clock.system(ZoneId.of("America/Sao_Paulo"))`.
- Testes substituem pelo `RelogioAjustavel` (subclasse de `Clock` em `src/test`, com `definir(OffsetDateTime)` e `avancar(Duration)`), registrado como `@Primary` em `RelogioTestConfig`.

### 4.4 Constantes de tempo (em `BilheteMapper`)
| Constante | Valor |
|-----------|-------|
| `OFFSET` | `ZoneOffset.of("-03:00")` |
| `FORMATO_SAIDA` | `DateTimeFormatter.ofPattern("uuuu-MM-dd'T'HH:mm:ssXXX")` |

- Datas nos DTOs são `String` já formatadas com `FORMATO_SAIDA`, garantindo sempre segundos e offset `-03:00` (evita `Z` e omissão de segundos do `toString()`).

---

## 5. Camada de Domínio

### 5.1 `StatusBilhete`
| Constante | `valorJson` |
|-----------|-------------|
| `ABERTO` | `"aberto"` |
| `ENCERRADO` | `"encerrado"` |
| `CANCELADO` | `"cancelado"` |

- Persistido com `@Enumerated(EnumType.STRING)`; exposto na API pelo `valorJson`.

### 5.2 `Bilhete` (entidade JPA)
Campos conforme seção 9 da constituição: `id`, `placa`, `entrada`, `saida`, `minutos`, `valorCentavos`, `status`.

- `@Id @GeneratedValue(strategy = IDENTITY)` → ids sequenciais a partir de 1.
- Índices em `placa` e em `status`.
- Construtor protegido para JPA + fábrica estática `Bilhete.abrir(placa, entrada)`.
- Toda `OffsetDateTime` é normalizada para `-03:00` antes de ser atribuída.

| Método de domínio | Pré-condição | Violação |
|-------------------|--------------|----------|
| `encerrar(saida, minutos, valorCentavos)` | `status == ABERTO` | `BilheteJaEncerradoException` |
| `cancelar()` | `status == ABERTO` | `BilheteNaoAbertoException` |

---

## 6. Camada de Repositório

`BilheteRepository extends JpaRepository<Bilhete, Long>`, apenas com query methods derivados (sem `@Query` com regra de negócio):

| Método | Uso |
|--------|-----|
| `existsByPlacaAndStatus(placa, status)` | UC8 — placa ocupada |
| `findByStatusOrderByEntradaDescIdDesc(status)` | UC3 — ativos |
| `findByPlacaOrderByEntradaDescIdDesc(placa)` | UC6 — histórico |
| `findByStatusAndSaidaGreaterThanEqualAndSaidaLessThan(status, inicio, fim)` | UC4 — relatório |

---

## 7. Camada de Serviço

### 7.1 `ValidadorEntrada`
| Método | Comportamento |
|--------|---------------|
| `validarPlaca(String)` | `null` ou não casa `^[A-Z0-9]{7}$` → `PlacaInvalidaException`. Sem `trim`/`toUpperCase`. |
| `parseEntrada(String)` | `null` → retorna `null` (usar agora). Senão `OffsetDateTime.parse`; `DateTimeParseException` → `EntradaInvalidaException`. Resultado convertido para `-03:00` e truncado em segundos. |
| `parseData(String)` | `null` → `DataInvalidaException`. Parse com padrão `uuuu-MM-dd` e `ResolverStyle.STRICT`; falha → `DataInvalidaException`. |

### 7.2 `CalculadoraTarifa` (componente puro, sem Spring Web/JPA)
Recebe `EstacionamentoProperties` pelo construtor (permite instanciar manualmente em teste unitário).

**`calcularMinutos(entrada, saida)`**: `Duration.between(entrada, saida).toMinutes()`, com mínimo `0`.

**`calcularValor(minutos)`** — passos, todos em `long`:

| Passo | Operação | Regra |
|-------|----------|-------|
| 1 | Se `minutos <= TOLERANCIA_MINUTOS` → retorna `0` | RN-06 |
| 2 | `fracoes = (minutos + FRACAO - 1) / FRACAO` | RN-08 |
| 3 | `valorFracao = TARIFA / (60 / FRACAO)` | RN-09 |
| 4 | Retorna `min(fracoes × valorFracao, TETO)` | RN-10 |

### 7.3 `BilheteService`
| Método | UC | Fluxo |
|---|---|---|
| `abrir(AbrirBilheteRequest)` | UC1, UC8 | `@Transactional`, `synchronized`. 1) `validarPlaca`; 2) `parseEntrada` (ou `now(clock)`); 3) `existsByPlacaAndStatus(placa, ABERTO)` → `BilheteEmAbertoException`; 4) `save(Bilhete.abrir(...))`; 5) mapear. |
| `encerrar(Long id)` | UC2, UC7 | `@Transactional`. 1) `findById` ou `BilheteNaoEncontradoException`; 2) `saida = now(clock)` truncado em segundos; 3) `calcularMinutos`; 4) `calcularValor`; 5) `bilhete.encerrar(...)` (valida status); 6) mapear para `EncerramentoResponse`. |
| `cancelar(Long id)` | UC5 | `@Transactional`. `findById` ou 404 → `bilhete.cancelar()` → mapear. |
| `listarAtivos()` | UC3 | `@Transactional(readOnly = true)`. `findByStatusOrderByEntradaDescIdDesc(ABERTO)`. |
| `historico(String placa)` | UC6 | `@Transactional(readOnly = true)`. `validarPlaca` → `findByPlacaOrderByEntradaDescIdDesc`. Lista vazia é resultado válido. |

Observação: o cálculo no `encerrar` é feito antes de `bilhete.encerrar(...)`, mas só é gravado se o status for `ABERTO`.

### 7.4 `RelatorioService`
`gerar(String data)` — `@Transactional(readOnly = true)`:
1. `dia = validador.parseData(data)`.
2. `inicio` = início do `dia` em `-03:00`; `fim` = início do dia seguinte em `-03:00`.
3. Busca encerrados com `saida` em `[inicio, fim)`.
4. `total` = quantidade; `faturamento` = soma de `valorCentavos`; `somaMin` = soma de `minutos`.
5. `tempoMedio` = `0` se `total == 0`; senão `(2 × somaMin + total) / (2 × total)` (0,5 para cima, inteiro — RN-19).
6. Retorna `RelatorioDiarioResponse` com `data` no formato `AAAA-MM-DD`.

### 7.5 `BilheteMapper`
| Método | Saída | UCs |
|--------|-------|-----|
| `toResponse(Bilhete)` | `BilheteResponse` | UC1, UC3, UC5, UC6 |
| `toEncerramento(Bilhete)` | `EncerramentoResponse` | UC2 |

Datas formatadas com `FORMATO_SAIDA`; status pelo `valorJson`.

---

## 8. DTOs

Todos como `record`. Campos JSON com `@JsonProperty` explícito em snake_case, além da estratégia global.

| DTO | Campos (nome JSON : tipo Java) | Observação |
|-----|-------------------------------|------------|
| `AbrirBilheteRequest` | `placa: String`, `entrada: String` | `entrada` como `String` para o parse manual gerar `entrada_invalida`. |
| `BilheteResponse` | `id: Long`, `placa: String`, `entrada: String`, `saida: String`, `minutos: Long`, `valor_centavos: Long`, `status: String` | `@JsonInclude(NON_NULL)`: aberto/cancelado omitem `saida`, `minutos`, `valor_centavos`. |
| `EncerramentoResponse` | `id: Long`, `placa: String`, `entrada: String`, `saida: String`, `minutos: Long`, `valor_centavos: Long` | Sem `status`, exatamente como o contrato do UC2. |
| `RelatorioDiarioResponse` | `data: String`, `total_bilhetes: long`, `faturamento_centavos: long`, `tempo_medio_minutos: long` | Todos inteiros. |
| `ErroResponse` | `erro: String` | Única chave do corpo de erro. |

---

## 9. Camada de Controller

### 9.1 `BilheteController` — `@RequestMapping("/bilhetes")`
| Anotação | Método | Retorno |
|---|---|---|
| `@PostMapping` | `abrir(@RequestBody(required = false) AbrirBilheteRequest)` | `201` |
| `@PostMapping("/{id}/encerramento")` | `encerrar(@PathVariable Long id)` | `200` |
| `@PostMapping("/{id}/cancelamento")` | `cancelar(@PathVariable Long id)` | `200` |
| `@GetMapping("/ativos")` | `listarAtivos()` | `200` |
| `@GetMapping(params = "placa")` | `historico(@RequestParam String placa)` | `200` |
| `@GetMapping` (sem `placa`) | `historicoSemPlaca()` | lança `PlacaInvalidaException` |

- Body nulo → service recebe `null` e lança `placa_invalida`.
- Não há `GET /bilhetes/{id}`, então `/ativos` não conflita com path variable.

### 9.2 `RelatorioController` — `@RequestMapping("/relatorios")`
| Anotação | Método | Retorno |
|---|---|---|
| `@GetMapping("/diario")` | `diario(@RequestParam(required = false) String data)` | `200` |

- Ausência e formato inválido caem no mesmo `DataInvalidaException` do validador.

---

## 10. Tratamento de Erros

### 10.1 Hierarquia
Classe abstrata `ApiException extends RuntimeException` com atributos `status` (`HttpStatus`) e `codigo` (`String`).

| Exceção | Status | Código |
|---|---|---|
| `PlacaInvalidaException` | 422 | `placa_invalida` |
| `EntradaInvalidaException` | 422 | `entrada_invalida` |
| `DataInvalidaException` | 422 | `data_invalida` |
| `BilheteNaoEncontradoException` | 404 | `bilhete_nao_encontrado` |
| `BilheteJaEncerradoException` | 409 | `bilhete_ja_encerrado` |
| `BilheteNaoAbertoException` | 409 | `bilhete_nao_aberto` |
| `BilheteEmAbertoException` | 409 | `bilhete_em_aberto` |

### 10.2 `GlobalExceptionHandler` (`@RestControllerAdvice`)
| Exceção capturada | Resposta |
|---|---|
| `ApiException` | `status` + `{"erro": codigo}` |
| `HttpMessageNotReadableException` | `422 placa_invalida` (JSON malformado no `POST /bilhetes`) |
| `MethodArgumentTypeMismatchException` (`id` não numérico) | `404 bilhete_nao_encontrado` |
| `MissingServletRequestParameterException` | `placa` → `422 placa_invalida`; `data` → `422 data_invalida` |
| `HttpMediaTypeNotSupportedException` | `422 placa_invalida` |
| `Exception` | `500 {"erro": "erro_interno"}` + log de erro |

- Todas as respostas com `Content-Type: application/json`.

---

## 11. Concorrência

- `BilheteService.abrir` é `synchronized` e `@Transactional`, garantindo que a checagem de placa ocupada e o `save` não intercalem entre requisições (instância única, banco em memória).
- `encerrar` e `cancelar` dependem do status lido na mesma transação; risco de corrida aceito no escopo (sem `@Version`).

---

## 12. Fluxos por Caso de Uso

| UC | Fluxo |
|----|-------|
| UC1 / UC8 — Abrir | `POST /bilhetes` → valida placa → parse entrada → checa aberto existente (409) → salva → `201 BilheteResponse` |
| UC2 / UC7 — Encerrar | `POST /bilhetes/{id}/encerramento` → busca (404) → `saida = now(clock)` → minutos → valor (tolerância → frações → teto) → `bilhete.encerrar` (409) → `200 EncerramentoResponse` |
| UC3 — Ativos | `GET /bilhetes/ativos` → abertos ordenados → `200 [BilheteResponse]` |
| UC4 — Relatório | `GET /relatorios/diario?data=` → parse estrito (422) → janela do dia em `-03:00` → agrega → `200 RelatorioDiarioResponse` |
| UC5 — Cancelar | `POST /bilhetes/{id}/cancelamento` → busca (404) → `bilhete.cancelar` (409) → `200 BilheteResponse` com `status: "cancelado"` |
| UC6 — Histórico | `GET /bilhetes?placa=` → valida placa (422) → bilhetes da placa ordenados → `200 [BilheteResponse]` (pode ser `[]`) |

---

## 13. Estratégia de Testes (resumo — detalhes em `tests.md`)

| Nível | Alvo | Ferramenta | Foco |
|---|---|---|---|
| Unitário | `CalculadoraTarifa` | JUnit 5 + AssertJ | Tolerância (limite e limite+1), fração exata e +1 min, teto, tolerância 0, minutos 0. |
| Unitário | `ValidadorEntrada` | JUnit 5 | Placas válidas/inválidas, ISO com e sem fuso, datas estritas (`2026-02-30`). |
| Unitário | `EstacionamentoProperties` | `ApplicationContextRunner` | Contexto falha com parâmetros inválidos. |
| Integração | Controllers | `@SpringBootTest` + MockMvc + `RelogioAjustavel` | Um teste por CA do `spec.md`; status HTTP, nomes exatos dos campos, ausência de campos extras nos erros, valores inteiros, offset `-03:00`. |

- Tempo simulado via campo `entrada` + `RelogioAjustavel`; proibido `Thread.sleep`.
- Banco limpo entre testes (`repository.deleteAll()` em `@BeforeEach`).

---

## 14. Ordem de Implementação (detalhes em `tasks.md`)

1. **Fundação**: projeto Maven, `pom.xml`, `application.properties`, `EstacionamentoProperties`, `ClockConfig`, `RelogioAjustavel`.
2. **Domínio e persistência**: `StatusBilhete`, `Bilhete`, `BilheteRepository`.
3. **Erros**: `ApiException`, exceções de domínio, `ErroResponse`, `GlobalExceptionHandler`.
4. **Regras puras**: `ValidadorEntrada`, `CalculadoraTarifa` + testes unitários.
5. **UC1 + UC8**: DTOs, mapper, `abrir`, endpoint `POST /bilhetes` + testes.
6. **UC2 + UC7**: `encerrar`, endpoint de encerramento + testes.
7. **UC5**: `cancelar`, endpoint de cancelamento + testes.
8. **UC3 + UC6**: listagens + testes.
9. **UC4**: `RelatorioService`, `RelatorioController` + testes.
10. **Validação final**: `./mvnw test` verde, subir na `PORTA_SERVICO` e rodar a suíte de correção.

---

## 15. Rastreabilidade (UC → Componentes)

| UC  | Controller | Service | Outros |
|-----|------------|---------|--------|
| UC1 | `BilheteController.abrir` | `BilheteService.abrir` | `ValidadorEntrada`, `BilheteMapper` |
| UC2 | `BilheteController.encerrar` | `BilheteService.encerrar` | `CalculadoraTarifa`, `Bilhete.encerrar` |
| UC3 | `BilheteController.listarAtivos` | `BilheteService.listarAtivos` | `BilheteRepository` |
| UC4 | `RelatorioController.diario` | `RelatorioService.gerar` | `ValidadorEntrada.parseData` |
| UC5 | `BilheteController.cancelar` | `BilheteService.cancelar` | `Bilhete.cancelar` |
| UC6 | `BilheteController.historico` | `BilheteService.historico` | `ValidadorEntrada.validarPlaca` |
| UC7 | — | `CalculadoraTarifa.calcularValor` | `EstacionamentoProperties` |
| UC8 | `BilheteController.abrir` | `BilheteService.abrir` | `BilheteRepository.existsByPlacaAndStatus` |

---

## 16. Riscos e Decisões Técnicas

| Item | Decisão | Motivo |
|---|---|---|
| Formato das datas | DTO com `String` formatada por `uuuu-MM-dd'T'HH:mm:ssXXX` | `OffsetDateTime.toString()` omite segundos zerados e o Jackson pode serializar em UTC. |
| `entrada` como `String` no request | Parse manual | Garante `422 entrada_invalida` em vez de `400` do Jackson. |
| Minutos | Truncados (`toMinutes()`) | Coerente com RN-05; requisição feita segundos após `entrada` simulada não pula para o minuto seguinte. |
| Média do relatório | `(2*soma + total) / (2*total)` | Arredondamento 0,5 para cima sem ponto flutuante. |
| Encerrar bilhete cancelado | `409 bilhete_ja_encerrado` | Contrato só define esse código para encerramento indevido. |
| Placa ausente no `GET /bilhetes` | `422 placa_invalida` | Coerente com CA-06.4. |
| Concorrência | `synchronized` em `abrir` | Suficiente para instância única com H2 em memória. |
| Relógio de teste | `RelogioAjustavel` em vez de `Clock.fixed` | Testes do relatório precisam encerrar bilhetes em dias diferentes. |