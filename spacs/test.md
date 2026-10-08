# Tests — API de Estacionamento Rotativo (Bilhetes)

Plano de testes derivado dos Critérios de Aceite do `spec.md`, seguindo a estratégia da seção 13 do `plan.md`.
Subordinado ao `constitution.md`. Todo CA do `spec.md` DEVE ter ao menos um teste correspondente.
Build só é aceito com `./mvnw test` verde.

## 1. Convenções

### 1.1 Notação dos parâmetros da variante
| Símbolo | Significado |
|---------|-------------|
| `T`     | `TARIFA_HORA_CENTAVOS` |
| `F`     | `FRACAO_MINUTOS` |
| `TETO`  | `TETO_DIARIO_CENTAVOS` |
| `TOL`   | `TOLERANCIA_MINUTOS` |
| `VF`    | Valor da fração = `T / (60 / F)` |
| `K`     | Nº de frações seguramente acima da tolerância = `(TOL / F) + 2` |
| `M_TETO`| Minutos que garantidamente estouram o teto = `((TETO / VF) + 1) * F` |

Todas as divisões são inteiras. Valores esperados são calculados com esses símbolos, nunca hardcoded.
Os testes leem os parâmetros de `EstacionamentoProperties` injetado, para continuarem válidos se a variante mudar.

### 1.2 Relógio
- Instante de referência: `AGORA = 2026-10-07T12:00:00-03:00`.
- Testes de integração usam um `RelogioAjustavel` (subclasse de `Clock` em `src/test`) registrado como `@Primary` em `@TestConfiguration`, com métodos `definir(OffsetDateTime)` e `avancar(Duration)`.
- Para simular permanência de `N` minutos: abrir com `entrada = AGORA - N min` e encerrar com o relógio em `AGORA` → `minutos = N`.
- Proibido `Thread.sleep`.

### 1.3 Isolamento
- `@BeforeEach`: `repository.deleteAll()` e `relogio.definir(AGORA)`.
- Ids não são assumidos como fixos entre testes; sempre usar o `id` retornado pela abertura.

### 1.4 Asserções de contrato (aplicadas em todos os testes de integração)
- `Content-Type` = `application/json`.
- Nomes de campos exatos em snake_case.
- Datas casam com regex `^\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}-03:00$`.
- Campos monetários e de minutos são inteiros (`jsonPath(...).value(instanceOf(Number.class))` + body bruto sem ponto decimal nesses campos).
- Corpo de erro tem **exatamente uma chave** (`jsonPath("$.*", hasSize(1))`) com o código esperado.

## 2. Classes de Teste

| Classe | Tipo | Alvo |
|--------|------|------|
| `CalculadoraTarifaTest` | Unitário | `CalculadoraTarifa` |
| `ValidadorEntradaTest` | Unitário | `ValidadorEntrada` |
| `EstacionamentoPropertiesTest` | Unitário (`ApplicationContextRunner`) | Validação dos parâmetros na inicialização |
| `BilheteControllerIT` | Integração (`@SpringBootTest` + `@AutoConfigureMockMvc`) | UC1, UC2, UC3, UC5, UC6, UC7, UC8 |
| `RelatorioControllerIT` | Integração (`@SpringBootTest` + `@AutoConfigureMockMvc`) | UC4 |
| `ErrosGeraisIT` | Integração | JSON malformado, id não numérico, formato do erro |


## 3. Testes Unitários

### 3.1 `CalculadoraTarifaTest` — cálculo de valor

| ID | Cenário | Entrada (`minutos`) | Esperado (`valor_centavos`) | Ref |
|----|---------|---------------------|-----------------------------|-----|
| UT-CALC-01 | Zero minutos | `0` | `0` | CA-07.1 |
| UT-CALC-02 | Exatamente na tolerância | `TOL` | `0` | CA-07.1, RN-06 |
| UT-CALC-03 | Tolerância + 1 cobra desde o 1º minuto | `TOL + 1` | `min(ceil((TOL+1)/F) * VF, TETO)` | CA-07.2, RN-07 |
| UT-CALC-04 | Fração exata | `K * F` | `min(K * VF, TETO)` | CA-02.2, RN-08 |
| UT-CALC-05 | Fração exata + 1 minuto | `K * F + 1` | `min((K + 1) * VF, TETO)` | CA-02.3, RN-08 |
| UT-CALC-06 | Fração exata − 1 minuto | `K * F - 1` | `min(K * VF, TETO)` | RN-08 |
| UT-CALC-07 | Estoura o teto | `M_TETO` | `TETO` | CA-02.4, RN-10 |
| UT-CALC-08 | Muito acima do teto (24h) | `1440` | `TETO` | RN-10 |
| UT-CALC-09 | Valor nunca excede o teto | laço `0..1440` | `valor <= TETO` em todos | RN-10 |
| UT-CALC-10 | Monotonicidade | laço `0..1440` | `valor(m+1) >= valor(m)` para `m > TOL` | RN-08 |

Testes com **properties construídas manualmente** (independem da variante):

| ID | Properties (`T`, `F`, `TETO`, `TOL`) | `minutos` | Esperado | Ref |
|----|--------------------------------------|-----------|----------|-----|
| UT-CALC-11 | `600, 15, 4000, 0` | `1` | `150` | CA-07.3 |
| UT-CALC-12 | `600, 15, 4000, 0` | `15` | `150` | fração exata |
| UT-CALC-13 | `600, 15, 4000, 0` | `16` | `300` | fração + 1 |
| UT-CALC-14 | `600, 15, 4000, 10` | `10` | `0` | tolerância |
| UT-CALC-15 | `600, 15, 4000, 10` | `11` | `150` | não desconta tolerância |
| UT-CALC-16 | `600, 15, 4000, 15` | `95` | `1050` | 7 frações × 150 |
| UT-CALC-17 | `600, 15, 1000, 0` | `120` | `1000` | teto (bruto 1200) |
| UT-CALC-18 | `500, 10, 9999, 0` | `61` | `583` | `VF = 500/6 = 83`; 7 × 83 |

UT-CALC-18 documenta o efeito da divisão inteira quando `T` não é divisível por `60/F`.

### 3.2 `CalculadoraTarifaTest` — cálculo de minutos

| ID | Entrada | Saída | Esperado | Ref |
|----|---------|-------|----------|-----|
| UT-MIN-01 | `12:00:00` | `12:00:00` | `0` | RN-05 |
| UT-MIN-02 | `10:25:00` | `12:00:00` | `95` | RN-05 |
| UT-MIN-03 | `11:59:01` | `12:00:00` | `0` (truncado) | RN-05 |
| UT-MIN-04 | `10:24:30` | `12:00:00` | `95` (truncado) | RN-05 |
| UT-MIN-05 | Entrada no futuro (`12:10`) | `12:00:00` | `0` | plan §7.2 |
| UT-MIN-06 | `2026-10-06T23:00-03:00` | `2026-10-07T01:30-03:00` | `150` (vira o dia) | RN-05 |
| UT-MIN-07 | Entrada em `Z` equivalente | `-03:00` | mesmo resultado do offset local | RN-03 |

### 3.3 `ValidadorEntradaTest`

**Placa**
| ID | Valor | Esperado | Ref |
|----|-------|----------|-----|
| UT-VAL-01 | `"ABC1D23"` | válido | RN-01 |
| UT-VAL-02 | `"ABC1234"` | válido | RN-01 |
| UT-VAL-03 | `"1234567"` | válido | RN-01 |
| UT-VAL-04 | `null` | `PlacaInvalidaException` | CA-01.3 |
| UT-VAL-05 | `""` | `PlacaInvalidaException` | CA-01.3 |
| UT-VAL-06 | `"abc1d23"` | `PlacaInvalidaException` | CA-01.3 |
| UT-VAL-07 | `"ABC1D2"` (6) | `PlacaInvalidaException` | CA-01.3 |
| UT-VAL-08 | `"ABC1D234"` (8) | `PlacaInvalidaException` | CA-01.3 |
| UT-VAL-09 | `"ABC-123"` | `PlacaInvalidaException` | CA-01.3 |
| UT-VAL-10 | `" ABC1D23"` / `"ABC1D23 "` | `PlacaInvalidaException` | RN-01 (sem trim) |
| UT-VAL-11 | `"ÁBC1D23"` | `PlacaInvalidaException` | RN-01 |

**Entrada**
| ID | Valor | Esperado | Ref |
|----|-------|----------|-----|
| UT-VAL-12 | `null` | retorna `null` (usar agora) | RF-02 |
| UT-VAL-13 | `"2026-10-07T10:00:00-03:00"` | mesmo instante, offset `-03:00` | CA-01.2 |
| UT-VAL-14 | `"2026-10-07T13:00:00Z"` | `2026-10-07T10:00:00-03:00` | CA-01.2, RN-03 |
| UT-VAL-15 | `"2026-10-07T10:00:00.789-03:00"` | truncado para `10:00:00` | plan §7.1 |
| UT-VAL-16 | `"2026-10-07T10:00:00"` (sem fuso) | `EntradaInvalidaException` | CA-01.4 |
| UT-VAL-17 | `"2026-10-07 10:00"` | `EntradaInvalidaException` | CA-01.4 |
| UT-VAL-18 | `"ontem"` | `EntradaInvalidaException` | CA-01.4 |
| UT-VAL-19 | `""` | `EntradaInvalidaException` | CA-01.4 |
| UT-VAL-20 | `"2026-02-30T10:00:00-03:00"` | `EntradaInvalidaException` | CA-01.4 |

**Data do relatório**
| ID | Valor | Esperado | Ref |
|----|-------|----------|-----|
| UT-VAL-21 | `"2026-10-07"` | `LocalDate 2026-10-07` | RN-20 |
| UT-VAL-22 | `null` | `DataInvalidaException` | CA-04.5 |
| UT-VAL-23 | `"07/10/2026"` | `DataInvalidaException` | CA-04.5 |
| UT-VAL-24 | `"2026-02-30"` | `DataInvalidaException` | CA-04.5 |
| UT-VAL-25 | `"2026-2-5"` | `DataInvalidaException` | RN-20 |
| UT-VAL-26 | `"2026-10-07T00:00:00"` | `DataInvalidaException` | RN-20 |
| UT-VAL-27 | `"2028-02-29"` (bissexto) | válido | RN-20 |

### 3.4 `EstacionamentoPropertiesTest`

| ID | Configuração | Esperado | Ref |
|----|--------------|----------|-----|
| UT-CFG-01 | Todos os parâmetros válidos | contexto sobe | constitution §2 |
| UT-CFG-02 | `fracao-minutos` ausente | contexto falha | constitution §2 |
| UT-CFG-03 | `fracao-minutos=7` (não divide 60) | contexto falha | constitution §2 |
| UT-CFG-04 | `tolerancia-minutos=5` | contexto falha | constitution §2 |
| UT-CFG-05 | `tarifa-hora-centavos=0` | contexto falha | constitution §2 |
| UT-CFG-06 | `teto-diario-centavos=-1` | contexto falha | constitution §2 |

## 4. Testes de Integração — `BilheteControllerIT`

### 4.1 UC1 — Abrir bilhete

| ID | Requisição | Esperado | Ref |
|----|-----------|----------|-----|
| IT-UC1-01 | `POST /bilhetes {"placa":"ABC1D23"}` | `201`; `id` numérico; `placa:"ABC1D23"`; `entrada` = `AGORA` em `-03:00`; `status:"aberto"`; sem `saida`/`minutos`/`valor_centavos` | CA-01.1 |
| IT-UC1-02 | `{"placa":"ABC1D23","entrada":"2026-10-07T10:00:00-03:00"}` | `201`; `entrada:"2026-10-07T10:00:00-03:00"` | CA-01.2 |
| IT-UC1-03 | `{"placa":"ABC1D23","entrada":"2026-10-07T13:00:00Z"}` | `201`; `entrada:"2026-10-07T10:00:00-03:00"` | CA-01.2, RN-03 |
| IT-UC1-04 | Duas aberturas de placas diferentes | ids distintos e crescentes | RF-01 |
| IT-UC1-05 | `{}` | `422 {"erro":"placa_invalida"}` | CA-01.3 |
| IT-UC1-06 | `{"placa":"abc1d23"}` | `422 placa_invalida` | CA-01.3 |
| IT-UC1-07 | `{"placa":"ABC1D2"}` | `422 placa_invalida` | CA-01.3 |
| IT-UC1-08 | `{"placa":"ABC1D234"}` | `422 placa_invalida` | CA-01.3 |
| IT-UC1-09 | `{"placa":"ABC-123"}` | `422 placa_invalida` | CA-01.3 |
| IT-UC1-10 | `{"placa":null}` | `422 placa_invalida` | CA-01.3 |
| IT-UC1-11 | `{"placa":1234567}` (número) | `422 placa_invalida` | CA-01.3 |
| IT-UC1-12 | `{"placa":"ABC1D23","entrada":"ontem"}` | `422 {"erro":"entrada_invalida"}` | CA-01.4 |
| IT-UC1-13 | `{"placa":"ABC1D23","entrada":"2026-10-07 10:00"}` | `422 entrada_invalida` | CA-01.4 |
| IT-UC1-14 | `{"placa":"ABC1D23","entrada":"2026-10-07T10:00:00"}` | `422 entrada_invalida` | CA-01.4 |
| IT-UC1-15 | Placa inválida **e** entrada inválida | `422 placa_invalida` (placa validada primeiro) | constitution §7.2 |
| IT-UC1-16 | Entrada inválida não cria bilhete | após IT-UC1-12, `GET /bilhetes/ativos` → `[]` | RN-02 |

### 4.2 UC2 — Encerrar bilhete

| ID | Preparação | Requisição | Esperado | Ref |
|----|-----------|-----------|----------|-----|
| IT-UC2-01 | Abrir com `entrada = AGORA - 95min` | `POST /bilhetes/{id}/encerramento` | `200`; campos `id`, `placa`, `entrada`, `saida`, `minutos`, `valor_centavos` (e **somente** esses); `saida` = `AGORA`; `minutos:95`; `valor_centavos` = `calcularValor(95)` | CA-02.1 |
| IT-UC2-02 | `entrada = AGORA - K*F min` | encerrar | `minutos = K*F`; `valor_centavos = min(K*VF, TETO)` | CA-02.2 |
| IT-UC2-03 | `entrada = AGORA - (K*F + 1) min` | encerrar | `valor_centavos = min((K+1)*VF, TETO)` | CA-02.3 |
| IT-UC2-04 | `entrada = AGORA - M_TETO min` | encerrar | `valor_centavos = TETO` | CA-02.4 |
| IT-UC2-05 | `entrada = AGORA - 3 dias` | encerrar | `valor_centavos = TETO` | CA-02.4 |
| IT-UC2-06 | Bilhete já encerrado | encerrar de novo | `409 {"erro":"bilhete_ja_encerrado"}` | CA-02.5 |
| IT-UC2-07 | Bilhete cancelado | encerrar | `409 bilhete_ja_encerrado` | CA-02.5, RN-12 |
| IT-UC2-08 | — | `POST /bilhetes/999999/encerramento` | `404 {"erro":"bilhete_nao_encontrado"}` | CA-02.6 |
| IT-UC2-09 | Qualquer encerramento | — | body bruto sem `.` em `valor_centavos` e `minutos` | CA-02.7, RN-11 |
| IT-UC2-10 | Encerramento com erro 409 | — | estado do bilhete inalterado (`saida`/`valor` originais em `GET /bilhetes?placa=`) | RN-12 |
| IT-UC2-11 | Sem `entrada`; `relogio.avancar(30 min)` | encerrar | `minutos:30`; valor = `calcularValor(30)` | RN-05 |

### 4.3 UC7 — Tolerância gratuita

| ID | Preparação (`entrada`) | Esperado ao encerrar | Ref |
|----|-----------------------|----------------------|-----|
| IT-UC7-01 | `AGORA` (0 min) | `valor_centavos:0` | CA-07.1 |
| IT-UC7-02 | `AGORA - TOL min` | `valor_centavos:0` | CA-07.1 |
| IT-UC7-03 | `AGORA - (TOL + 1) min` | `valor_centavos = min(ceil((TOL+1)/F) * VF, TETO)` > 0 | CA-07.2 |
| IT-UC7-04 | `AGORA - (TOL + 1) min` | valor **igual** ao que seria cobrado sem tolerância para `TOL+1` minutos (não desconta) | RN-07 |
| IT-UC7-05 | Somente se `TOL = 0`: `AGORA - 1 min` | `valor_centavos = VF` | CA-07.3 |

> IT-UC7-05 usa `assumeTrue(props.toleranciaMinutos() == 0)`; o comportamento com tolerância 0 é coberto sempre por UT-CALC-11.

### 4.4 UC3 — Listar ativos

| ID | Preparação | Requisição | Esperado | Ref |
|----|-----------|-----------|----------|-----|
| IT-UC3-01 | Abrir A (aberto), B (encerrado), C (cancelado) | `GET /bilhetes/ativos` | `200`; array só com A | CA-03.1 |
| IT-UC3-02 | Abrir P1 `entrada 09:00`, P2 `entrada 11:00`, P3 `entrada 10:00` | `GET /bilhetes/ativos` | ordem: P2, P3, P1 | CA-03.2, RN-16 |
| IT-UC3-03 | Duas placas com mesma `entrada` | `GET /bilhetes/ativos` | maior `id` primeiro | RN-16 |
| IT-UC3-04 | Nenhum bilhete | `GET /bilhetes/ativos` | `200 []` | CA-03.3 |
| IT-UC3-05 | Itens do array | — | contêm `id`, `placa`, `entrada`, `status:"aberto"`; sem `saida`/`valor_centavos` | RN-03 |

### 4.5 UC5 — Cancelar bilhete

| ID | Preparação | Requisição | Esperado | Ref |
|----|-----------|-----------|----------|-----|
| IT-UC5-01 | Bilhete aberto | `POST /bilhetes/{id}/cancelamento` | `200`; `status:"cancelado"`; sem `saida`, `minutos`, `valor_centavos` | CA-05.1, RN-14 |
| IT-UC5-02 | Bilhete encerrado | cancelar | `409 {"erro":"bilhete_nao_aberto"}` | CA-05.2 |
| IT-UC5-03 | Bilhete já cancelado | cancelar de novo | `409 bilhete_nao_aberto` | CA-05.2 |
| IT-UC5-04 | — | `POST /bilhetes/999999/cancelamento` | `404 bilhete_nao_encontrado` | CA-05.3 |
| IT-UC5-05 | Bilhete cancelado | `GET /bilhetes/ativos` | não aparece | CA-05.4 |
| IT-UC5-06 | Bilhete cancelado hoje | `GET /relatorios/diario?data=2026-10-07` | não conta em `total_bilhetes` | CA-05.4 |

### 4.6 UC6 — Histórico por placa

| ID | Preparação | Requisição | Esperado | Ref |
|----|-----------|-----------|----------|-----|
| IT-UC6-01 | `ABC1D23`: 1 encerrado (`entrada 08:00`), 1 cancelado (`09:00`), 1 aberto (`10:00`) | `GET /bilhetes?placa=ABC1D23` | `200`; 3 itens, ordem aberto → cancelado → encerrado | CA-06.1, RN-16 |
| IT-UC6-02 | Item encerrado do histórico | — | contém `saida`, `minutos`, `valor_centavos`, `status:"encerrado"` | RF-07 |
| IT-UC6-03 | Item cancelado do histórico | — | `status:"cancelado"`, sem `saida`/`valor_centavos` | RN-14 |
| IT-UC6-04 | Bilhetes de `ABC1D23` e `XYZ9W87` | `GET /bilhetes?placa=ABC1D23` | nenhum item de `XYZ9W87` | CA-06.2 |
| IT-UC6-05 | — | `GET /bilhetes?placa=ZZZ9Z99` | `200 []` | CA-06.3, RN-17 |
| IT-UC6-06 | — | `GET /bilhetes?placa=abc1d23` | `422 placa_invalida` | CA-06.4 |
| IT-UC6-07 | — | `GET /bilhetes?placa=ABC` | `422 placa_invalida` | CA-06.4 |
| IT-UC6-08 | — | `GET /bilhetes` (sem parâmetro) | `422 placa_invalida` | plan §9.1 |

### 4.7 UC8 — Uma vaga por placa

| ID | Preparação | Requisição | Esperado | Ref |
|----|-----------|-----------|----------|-----|
| IT-UC8-01 | `ABC1D23` aberto | `POST /bilhetes {"placa":"ABC1D23"}` | `409 {"erro":"bilhete_em_aberto"}` | CA-08.1 |
| IT-UC8-02 | Após IT-UC8-01 | `GET /bilhetes?placa=ABC1D23` | apenas 1 bilhete (o 409 não criou outro) | RN-04 |
| IT-UC8-03 | `ABC1D23` aberto → encerrado | abrir de novo | `201`; novo `id` | CA-08.2 |
| IT-UC8-04 | `ABC1D23` aberto → cancelado | abrir de novo | `201`; novo `id` | CA-08.3 |
| IT-UC8-05 | `ABC1D23` aberto | abrir `XYZ9W87` | `201` (outra placa não é afetada) | RN-04 |
| IT-UC8-06 | 10 threads abrindo `ABC1D23` simultaneamente | — | exatamente 1 × `201` e 9 × `409`; 1 bilhete aberto | plan §11 |
| IT-UC8-07 | `ABC1D23` aberto | `POST` com placa ocupada **e** `entrada` inválida | `422 entrada_invalida` (validação antes do conflito) | constitution §7.2 |

---

## 5. Testes de Integração — `RelatorioControllerIT`

Preparação base: relógio em `AGORA` (2026-10-07 12:00 -03:00).

| ID | Preparação | Requisição | Esperado | Ref |
|----|-----------|-----------|----------|-----|
| IT-UC4-01 | Encerrar 3 bilhetes hoje com 30, 60 e 90 min | `GET /relatorios/diario?data=2026-10-07` | `200`; `data:"2026-10-07"`; `total_bilhetes:3`; `faturamento_centavos` = soma de `calcularValor(30/60/90)`; `tempo_medio_minutos:60` | CA-04.1 |
| IT-UC4-02 | Encerrados de 10 e 11 min | idem | `tempo_medio_minutos:11` (10,5 → 11) | CA-04.3, RN-19 |
| IT-UC4-03 | Encerrados de 10, 10 e 11 min | idem | `tempo_medio_minutos:10` (10,33 → 10) | RN-19 |
| IT-UC4-04 | Encerrados de 10, 11, 11 min | idem | `tempo_medio_minutos:11` (10,67 → 11) | RN-19 |
| IT-UC4-05 | 1 aberto, 1 cancelado, 1 encerrado hoje | idem | `total_bilhetes:1` | CA-04.2 |
| IT-UC4-06 | Encerrar 1 bilhete com relógio em `2026-10-06T12:00-03:00`, outro em `AGORA` | `data=2026-10-07` | `total_bilhetes:1` | CA-04.2 |
| IT-UC4-07 | Bilhete aberto `2026-10-06T23:00-03:00`, encerrado `2026-10-07T00:30-03:00` | `data=2026-10-07` | conta em 07 | RN-18 |
| IT-UC4-08 | Mesmo bilhete de IT-UC4-07 | `data=2026-10-06` | não conta em 06 | RN-18 |
| IT-UC4-09 | Encerrar com relógio em `2026-10-07T23:59:59-03:00` | `data=2026-10-07` | conta (limite superior do dia em -03:00) | RN-18 |
| IT-UC4-10 | Encerrar com relógio em `2026-10-08T00:00:00-03:00` | `data=2026-10-07` | não conta | RN-18 |
| IT-UC4-11 | Nenhum bilhete | `data=2026-10-07` | `200`; `total_bilhetes:0`; `faturamento_centavos:0`; `tempo_medio_minutos:0` | CA-04.4 |
| IT-UC4-12 | — | `GET /relatorios/diario` (sem `data`) | `422 {"erro":"data_invalida"}` | CA-04.5 |
| IT-UC4-13 | — | `data=07/10/2026` | `422 data_invalida` | CA-04.5 |
| IT-UC4-14 | — | `data=2026-02-30` | `422 data_invalida` | CA-04.5 |
| IT-UC4-15 | — | `data=` (vazio) | `422 data_invalida` | CA-04.5 |
| IT-UC4-16 | Encerrar um bilhete que estoura o teto + um de valor normal | `data=2026-10-07` | `faturamento_centavos = TETO + calcularValor(normal)` | RN-10, RN-11 |
| IT-UC4-17 | Qualquer relatório | — | body bruto sem ponto decimal; todos os campos numéricos inteiros | RN-11 |

---

## 6. Testes de Integração — `ErrosGeraisIT`

| ID | Requisição | Esperado | Ref |
|----|-----------|----------|-----|
| IT-ERR-01 | `POST /bilhetes` com body `{placa:` (JSON malformado) | `422 placa_invalida` | plan §10.2 |
| IT-ERR-02 | `POST /bilhetes` sem body | `422 placa_invalida` | plan §10.2 |
| IT-ERR-03 | `POST /bilhetes` com `Content-Type: text/plain` | `422 placa_invalida` | plan §10.2 |
| IT-ERR-04 | `POST /bilhetes/abc/encerramento` | `404 bilhete_nao_encontrado` | RN-15 |
| IT-ERR-05 | `POST /bilhetes/abc/cancelamento` | `404 bilhete_nao_encontrado` | RN-15 |
| IT-ERR-06 | Todos os erros acima | body com exatamente a chave `erro`; sem `timestamp`, `path`, `trace`, `status` | constitution §8 |
| IT-ERR-07 | Todos os erros acima | `Content-Type: application/json` | constitution §5 |


