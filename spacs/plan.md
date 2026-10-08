# Plan — API de Estacionamento Rotativo (Bilhetes)

Plano técnico de implementação. Traduz o `spec.md` em decisões de código, obedecendo ao `constitution.md`.
Em conflito, prevalece: contrato do enunciado > `constitution.md` > `spec.md` > `plan.md`.
Detalhamento de casos de teste fica em `tests.md`; lista executável de tarefas fica em `tasks.md`.

## 1. Visão Geral da Solução

- Aplicação Spring Boot única, empacotada como JAR executável, escutando em `PORTA_SERVICO`.
- Arquitetura em camadas: `controller → service → repository → domain`.
- Persistência em H2 em memória via Spring Data JPA.
- Todo cálculo de tempo usa `Clock` injetado; todo cálculo monetário usa `long`.
- Erros tratados centralmente por `GlobalExceptionHandler`, sempre no formato `{"erro": "<codigo>"}`.

## 2. Estrutura do Projeto

```
estacionamento/
├── pom.xml
├── mvnw / mvnw.cmd / .mvn/
├── src/main/java/com/estacionamento/
│   ├── EstacionamentoApplication.java
│   ├── config/
│   │   ├── EstacionamentoProperties.java
│   │   └── ClockConfig.java
│   ├── controller/
│   │   ├── BilheteController.java
│   │   └── RelatorioController.java
│   ├── service/
│   │   ├── BilheteService.java
│   │   ├── RelatorioService.java
│   │   ├── CalculadoraTarifa.java
│   │   ├── ValidadorEntrada.java
│   │   └── BilheteMapper.java
│   ├── repository/
│   │   └── BilheteRepository.java
│   ├── domain/
│   │   ├── Bilhete.java
│   │   └── StatusBilhete.java
│   ├── dto/
│   │   ├── AbrirBilheteRequest.java
│   │   ├── BilheteResponse.java
│   │   ├── EncerramentoResponse.java
│   │   ├── RelatorioDiarioResponse.java
│   │   └── ErroResponse.java
│   └── exception/
│       ├── ApiException.java
│       ├── PlacaInvalidaException.java
│       ├── EntradaInvalidaException.java
│       ├── DataInvalidaException.java
│       ├── BilheteNaoEncontradoException.java
│       ├── BilheteJaEncerradoException.java
│       ├── BilheteNaoAbertoException.java
│       ├── BilheteEmAbertoException.java
│       └── GlobalExceptionHandler.java
├── src/main/resources/
│   └── application.properties
└── src/test/java/com/estacionamento/
    ├── service/CalculadoraTarifaTest.java
    ├── service/ValidadorEntradaTest.java
    ├── controller/BilheteControllerIT.java
    └── controller/RelatorioControllerIT.java
```



## 3. Dependências (`pom.xml`)

- `spring-boot-starter-parent` 3.x, `java.version` = 21.
- `spring-boot-starter-web`
- `spring-boot-starter-validation`
- `spring-boot-starter-data-jpa`
- `com.h2database:h2` (scope `runtime`)
- `spring-boot-starter-test` (scope `test`; já traz JUnit 5, MockMvc, AssertJ)
- `spring-boot-configuration-processor` (opcional, apenas para metadados das properties)

Nenhuma outra dependência (Lombok, MapStruct etc. proibidos pela constituição).



## 4. Configuração

### 4.1 `application.properties`
Conforme seção 10 da constituição: porta, parâmetros da variante, H2 em memória, `ddl-auto=create-drop`, `open-in-view=false`, `include-stacktrace=never`.

### 4.2 `EstacionamentoProperties`
```java
@Validated
@ConfigurationProperties(prefix = "estacionamento")
public record EstacionamentoProperties(
    @Positive long tarifaHoraCentavos,
    @Positive int fracaoMinutos,
    @Positive long tetoDiarioCentavos,
    @PositiveOrZero int toleranciaMinutos
) {
    @AssertTrue  
    public boolean isFracaoDivisorDe60() { return fracaoMinutos > 0 && 60 % fracaoMinutos == 0; }

    @AssertTrue  
    public boolean isToleranciaValida() { return toleranciaMinutos == 0 || toleranciaMinutos == 10 || toleranciaMinutos == 15; }
}
```
- Registrada com `@ConfigurationPropertiesScan` na classe principal.
- Falha de validação impede a aplicação de subir.

### 4.3 `ClockConfig`
```java
@Bean
public Clock clock() { return Clock.system(ZoneId.of("America/Sao_Paulo")); }
```
- Testes substituem por `Clock.fixed(...)` via `@TestConfiguration` / `@MockBean`.

### 4.4 Constantes de tempo (em `BilheteMapper` ou classe utilitária)
```java
ZoneOffset OFFSET = ZoneOffset.of("-03:00");
DateTimeFormatter FORMATO_SAIDA = DateTimeFormatter.ofPattern("uuuu-MM-dd'T'HH:mm:ssXXX");
```
- Datas nos DTOs são `String` já formatadas com esse formatter, garantindo sempre segundos e offset `-03:00` (evita `Z` e omissão de segundos do `toString()`).

---

## 5. Camada de Domínio

### 5.1 `StatusBilhete`
```java
public enum StatusBilhete {
    ABERTO("aberto"), ENCERRADO("encerrado"), CANCELADO("cancelado");
    private final String valorJson;
}
```
- Persistido com `@Enumerated(EnumType.STRING)`.
- Exposto na API pelo `valorJson` (minúsculo).

### 5.2 `Bilhete` (entidade JPA)
Campos conforme seção 9 da constituição: `id`, `placa`, `entrada`, `saida`, `minutos`, `valorCentavos`, `status`.

- `@Id @GeneratedValue(strategy = IDENTITY)` → ids sequenciais a partir de 1.
- `@Index` em `placa` e em `status`.
- Construtor protegido para JPA + fábrica estática `Bilhete.abrir(String placa, OffsetDateTime entrada)`.
- Métodos de domínio:
  - `encerrar(OffsetDateTime saida, long minutos, long valorCentavos)` → se `status != ABERTO`, lança `BilheteJaEncerradoException`.
  - `cancelar()` → se `status != ABERTO`, lança `BilheteNaoAbertoException`.
- Toda `OffsetDateTime` é normalizada para `-03:00` antes de ser atribuída.

---

## 6. Camada de Repositório

```java
public interface BilheteRepository extends JpaRepository<Bilhete, Long> {
    boolean existsByPlacaAndStatus(String placa, StatusBilhete status);
    List<Bilhete> findByStatusOrderByEntradaDescIdDesc(StatusBilhete status);
    List<Bilhete> findByPlacaOrderByEntradaDescIdDesc(String placa);
    List<Bilhete> findByStatusAndSaidaGreaterThanEqualAndSaidaLessThan(
        StatusBilhete status, OffsetDateTime inicio, OffsetDateTime fim);
}
```
- Apenas query methods derivados; sem `@Query` com regra de negócio.

---

## 7. Camada de Serviço

### 7.1 `ValidadorEntrada`
| Método                          | Comportamento                                                                 |
|---------------------------------|-------------------------------------------------------------------------------|
| `String validarPlaca(String)`   | `null` ou não casa `^[A-Z0-9]{7}$` → `PlacaInvalidaException`. Sem `trim`/`toUpperCase`. |
| `OffsetDateTime parseEntrada(String)` | `null` → retorna `null` (usar agora). Senão `OffsetDateTime.parse(valor)`; `DateTimeParseException` → `EntradaInvalidaException`. Resultado convertido para `-03:00` e truncado em segundos. |
| `LocalDate parseData(String)`   | `null` → `DataInvalidaException`. Parse com `DateTimeFormatter.ofPattern("uuuu-MM-dd").withResolverStyle(ResolverStyle.STRICT)`; falha → `DataInvalidaException`. |

### 7.2 `CalculadoraTarifa` (componente puro, sem Spring Web/JPA)
```java
public long calcularMinutos(OffsetDateTime entrada, OffsetDateTime saida) {
    long m = Duration.between(entrada, saida).toMinutes();
    return Math.max(m, 0);
}

public long calcularValor(long minutos) {
    if (minutos <= props.toleranciaMinutos()) return 0;                      
    long fracoes = (minutos + props.fracaoMinutos() - 1) / props.fracaoMinutos();
    long valorFracao = props.tarifaHoraCentavos() / (60 / props.fracaoMinutos());
    return Math.min(fracoes * valorFracao, props.tetoDiarioCentavos());      
```
- Recebe `EstacionamentoProperties` pelo construtor (permite instanciar manualmente em teste unitário).

### 7.3 `BilheteService`
| Método | UC | Fluxo |
|---|---|---|
| `abrir(AbrirBilheteRequest)` | UC1, UC8 | `@Transactional`, `synchronized`. 1) `validarPlaca`; 2) `parseEntrada` (ou `now(clock)`); 3) `existsByPlacaAndStatus(placa, ABERTO)` → `BilheteEmAbertoException`; 4) `save(Bilhete.abrir(...))`; 5) mapear. |
| `encerrar(Long id)` | UC2, UC7 | `@Transactional`. 1) `findById` ou `BilheteNaoEncontradoException`; 2) `saida = now(clock)` truncado em segundos; 3) `minutos = calculadora.calcularMinutos`; 4) `valor = calculadora.calcularValor`; 5) `bilhete.encerrar(...)` (valida status); 6) mapear para `EncerramentoResponse`. |
| `cancelar(Long id)` | UC5 | `@Transactional`. `findById` ou 404 → `bilhete.cancelar()` → mapear. |
| `listarAtivos()` | UC3 | `@Transactional(readOnly = true)`. `findByStatusOrderByEntradaDescIdDesc(ABERTO)`. |
| `historico(String placa)` | UC6 | `@Transactional(readOnly = true)`. `validarPlaca` → `findByPlacaOrderByEntradaDescIdDesc`. Lista vazia é resultado válido. |

Observação: a verificação de status ocorre **antes** de persistir qualquer alteração; o cálculo no `encerrar` é feito antes de `bilhete.encerrar(...)`, mas só é gravado se o status for `ABERTO`.

### 7.4 `RelatorioService`
`gerar(String data)` — `@Transactional(readOnly = true)`:
1. `LocalDate dia = validador.parseData(data)`.
2. `inicio = dia.atStartOfDay().atOffset(OFFSET)`; `fim = dia.plusDays(1).atStartOfDay().atOffset(OFFSET)`.
3. `bilhetes = findByStatusAndSaidaGreaterThanEqualAndSaidaLessThan(ENCERRADO, inicio, fim)`.
4. `total = bilhetes.size()`; `faturamento = soma(valorCentavos)`; `somaMin = soma(minutos)`.
5. `tempoMedio = total == 0 ? 0 : (2 * somaMin + total) / (2 * total)` (média com 0,5 para cima, inteiro — RN-19).
6. Retorna `RelatorioDiarioResponse(dia.toString(), total, faturamento, tempoMedio)`.

### 7.5 `BilheteMapper`
- `toResponse(Bilhete)` → `BilheteResponse` (UC1, UC3, UC5, UC6), formatando datas com `FORMATO_SAIDA` e status com `valorJson`.
- `toEncerramento(Bilhete)` → `EncerramentoResponse` (UC2).

---

## 8. DTOs

```java
public record AbrirBilheteRequest(String placa, String entrada) {}

@JsonInclude(JsonInclude.Include.NON_NULL)
public record BilheteResponse(
    Long id, String placa, String entrada, String saida,
    Long minutos, @JsonProperty("valor_centavos") Long valorCentavos, String status) {}

public record EncerramentoResponse(
    Long id, String placa, String entrada, String saida,
    Long minutos, @JsonProperty("valor_centavos") Long valorCentavos) {}

public record RelatorioDiarioResponse(
    String data,
    @JsonProperty("total_bilhetes") long totalBilhetes,
    @JsonProperty("faturamento_centavos") long faturamentoCentavos,
    @JsonProperty("tempo_medio_minutos") long tempoMedioMinutos) {}

public record ErroResponse(String erro) {}
```
- `entrada` do request é `String` para o parse manual gerar `entrada_invalida` em vez de erro genérico do Jackson.
- `@JsonProperty` explícito em snake_case, além da estratégia global, para não depender só da configuração.
- `EncerramentoResponse` não tem `status`, seguindo exatamente o contrato do UC2.

---

## 9. Camada de Controller

### 9.1 `BilheteController` — `@RequestMapping("/bilhetes")`
| Anotação | Método | Retorno |
|---|---|---|
| `@PostMapping` | `abrir(@RequestBody(required = false) AbrirBilheteRequest req)` | `ResponseEntity.status(201)` |
| `@PostMapping("/{id}/encerramento")` | `encerrar(@PathVariable Long id)` | `200` |
| `@PostMapping("/{id}/cancelamento")` | `cancelar(@PathVariable Long id)` | `200` |
| `@GetMapping("/ativos")` | `listarAtivos()` | `200` |
| `@GetMapping(params = "placa")` | `historico(@RequestParam String placa)` | `200` |
| `@GetMapping` (sem `placa`) | `historicoSemPlaca()` | lança `PlacaInvalidaException` |

- Body nulo (`required = false`) → service recebe `null` e lança `placa_invalida`.
- Não há `GET /bilhetes/{id}`, então `/ativos` não conflita com path variable.

### 9.2 `RelatorioController` — `@RequestMapping("/relatorios")`
| Anotação | Método | Retorno |
|---|---|---|
| `@GetMapping("/diario")` | `diario(@RequestParam(required = false) String data)` | `200` |

- `data` como `String` opcional: ausência e formato inválido caem no mesmo `DataInvalidaException` do validador.

---

## 10. Tratamento de Erros

### 10.1 Hierarquia
```java
public abstract class ApiException extends RuntimeException {
    private final HttpStatus status;
    private final String codigo;
}
```
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

**UC1 / UC8 — Abrir**
`POST /bilhetes` → Controller → `BilheteService.abrir` → valida placa → parse entrada → checa aberto existente (409) → salva → `201 BilheteResponse`.

**UC2 / UC7 — Encerrar**
`POST /bilhetes/{id}/encerramento` → busca (404) → `saida = now(clock)` → minutos → valor (tolerância → frações → teto) → `bilhete.encerrar` (409 se não aberto) → `200 EncerramentoResponse`.

**UC3 — Ativos**
`GET /bilhetes/ativos` → `findByStatusOrderByEntradaDescIdDesc(ABERTO)` → `200 [BilheteResponse]`.

**UC4 — Relatório**
`GET /relatorios/diario?data=` → parse estrito (422) → janela `[dia 00:00-03:00, dia+1 00:00-03:00)` → encerrados na janela → agrega → `200 RelatorioDiarioResponse`.

**UC5 — Cancelar**
`POST /bilhetes/{id}/cancelamento` → busca (404) → `bilhete.cancelar` (409 se não aberto) → `200 BilheteResponse` com `status: "cancelado"`.

**UC6 — Histórico**
`GET /bilhetes?placa=` → valida placa (422) → `findByPlacaOrderByEntradaDescIdDesc` → `200 [BilheteResponse]` (pode ser `[]`).

---

## 13. Estratégia de Testes (resumo — detalhes em `tests.md`)

| Nível | Alvo | Ferramenta | Foco |
|---|---|---|---|
| Unitário | `CalculadoraTarifa` | JUnit 5 + AssertJ | Tolerância (limite e limite+1), fração exata e +1 min, teto, tolerância 0, minutos 0. |
| Unitário | `ValidadorEntrada` | JUnit 5 | Placas válidas/inválidas, ISO com e sem fuso, datas estritas (`2026-02-30`). |
| Integração | Controllers | `@SpringBootTest` + MockMvc + `Clock` fixo | Um teste por CA do `spec.md`; status HTTP, nomes exatos dos campos, ausência de campos extras nos erros, valores inteiros, offset `-03:00`. |

- Tempo simulado via campo `entrada` + `Clock` fixo; proibido `Thread.sleep`.
- Banco limpo entre testes (`@DirtiesContext` ou `repository.deleteAll()` em `@BeforeEach`).

---

## 14. Ordem de Implementação (detalhes em `tasks.md`)

1. **Fundação**: projeto Maven, `pom.xml`, `application.properties`, `EstacionamentoProperties`, `ClockConfig`.
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