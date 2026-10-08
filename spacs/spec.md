# Spec — API de Estacionamento Rotativo (Bilhetes)

Especificação funcional derivada dos casos de uso UC1–UC8 do enunciado.
Subordinada ao `constitution.md`. Em conflito, prevalece o contrato do enunciado e depois a constituição.
Parâmetros da variante (`TARIFA_HORA_CENTAVOS`, `FRACAO_MINUTOS`, `TETO_DIARIO_CENTAVOS`, `TOLERANCIA_MINUTOS`, `PORTA_SERVICO`) são referenciados pelo nome; os valores estão na seção 2 da constituição.


## 1. Usuários

| Ator                     | Descrição                                                                 | Casos de uso        |
|--------------------------|---------------------------------------------------------------------------|---------------------|
| Operador do estacionamento | Atendente que registra entrada, saída e cancelamento de bilhetes por placa. | UC1, UC2, UC3, UC5, UC6, UC8 |
| Gestor da operadora      | Responsável pelo acompanhamento financeiro e operacional diário.          | UC4                 |
| Motorista (indireto)     | Dono do veículo; não acessa a API, mas é afetado pela cobrança e tolerância. | UC2, UC7          |
| Suíte de correção        | Cliente automatizado que valida o contrato usando o campo `entrada` opcional. | Todos           |


## 2. Histórias de Usuário

Como operador, quero abrir um bilhete informando a placa do veículo, para registrar o horário de entrada.
Como operador, quero encerrar um bilhete aberto, para saber quantos minutos o veículo ficou e quanto deve ser cobrado.
Como operador, quero ver os bilhetes abertos, dos mais recentes para os mais antigos, para saber quais veículos estão no estacionamento.
Como gestor, quero consultar o relatório de um dia específico, para conhecer a quantidade de bilhetes encerrados, o faturamento e o tempo médio de permanência.
Como operador, quero cancelar um bilhete aberto por engano, para que ele não gere cobrança.
Como operador, quero consultar todos os bilhetes de uma placa, para atender dúvidas do motorista sobre utilizações anteriores.
Como motorista, quero não pagar se ficar até `TOLERANCIA_MINUTOS` minutos, para poder entrar e sair rapidamente sem custo.
Como operador, quero ser impedido de abrir um segundo bilhete para uma placa que já está com bilhete aberto, para evitar cobrança duplicada.


## 3. Requisitos Funcionais

| ID    | Requisito                                                                                                   | UC  |
|-------|-------------------------------------------------------------------------------------------------------------|-----|
| RF-01 | `POST /bilhetes` cria bilhete com `placa` e status `aberto`, retornando `201` com `id`, `placa`, `entrada`, `status`. | UC1 |
| RF-02 | `POST /bilhetes` aceita `entrada` opcional (ISO-8601 com fuso); ausente, usa o instante atual.              | UC1 |
| RF-03 | `POST /bilhetes/{id}/encerramento` encerra o bilhete, registra `saida`, calcula `minutos` e `valor_centavos`, retornando `200`. | UC2 |
| RF-04 | `GET /bilhetes/ativos` retorna `200` com array dos bilhetes `aberto`, mais recentes primeiro.               | UC3 |
| RF-05 | `GET /relatorios/diario?data=AAAA-MM-DD` retorna `200` com `data`, `total_bilhetes`, `faturamento_centavos`, `tempo_medio_minutos`. | UC4 |
| RF-06 | `POST /bilhetes/{id}/cancelamento` muda o status para `cancelado` e retorna `200`, sem `saida` nem `valor_centavos`. | UC5 |
| RF-07 | `GET /bilhetes?placa=XXXXXXX` retorna `200` com array de todos os bilhetes da placa, qualquer status, mais recentes primeiro. | UC6 |
| RF-08 | O cálculo de valor aplica a tolerância de `TOLERANCIA_MINUTOS` antes da cobrança por fração.               | UC7 |
| RF-09 | `POST /bilhetes` recusa abertura para placa com bilhete `aberto`, retornando `409 bilhete_em_aberto`.       | UC8 |
| RF-10 | Todos os erros retornam body `{"erro": "<codigo>"}` com o status definido na tabela de erros da constituição. | Todos |
| RF-11 | O serviço escuta na porta `PORTA_SERVICO`.                                                                 | Todos |


## 4. Regras de Negócio

| ID    | Regra                                                                                                       | UC  |
|-------|-------------------------------------------------------------------------------------------------------------|-----|
| RN-01 | Placa válida: exatamente 7 caracteres alfanuméricos maiúsculos (`^[A-Z0-9]{7}$`). Fora disso → `422 placa_invalida`. | UC1, UC6 |
| RN-02 | `entrada` deve ser ISO-8601 com fuso. Formato inválido ou sem fuso → `422 entrada_invalida`.                | UC1 |
| RN-03 | Todas as datas retornadas usam offset `-03:00`.                                                             | Todos |
| RN-04 | Uma placa só pode ter um bilhete `aberto` por vez. Após encerrar ou cancelar, pode abrir novamente.         | UC8 |
| RN-05 | `minutos` = minutos completos entre `entrada` e `saida`.                                                    | UC2 |
| RN-06 | Se `minutos <= TOLERANCIA_MINUTOS`, `valor_centavos = 0`.                                                   | UC7 |
| RN-07 | Passada a tolerância (mesmo por 1 minuto), cobra-se desde o primeiro minuto; a tolerância não é descontada. | UC7 |
| RN-08 | Cobrança por fração de `FRACAO_MINUTOS`, arredondando para cima: `fracoes = ceil(minutos / FRACAO_MINUTOS)`. Fração exata cobra 1; 1 minuto a mais cobra a seguinte. | UC2 |
| RN-09 | Valor da fração = `TARIFA_HORA_CENTAVOS ÷ (60 ÷ FRACAO_MINUTOS)`.                                           | UC2 |
| RN-10 | `valor_centavos = min(fracoes × valorFracao, TETO_DIARIO_CENTAVOS)`.                                        | UC2 |
| RN-11 | Valores monetários sempre inteiros em centavos; a API nunca retorna ponto flutuante.                        | UC2, UC4 |
| RN-12 | Só bilhete `aberto` pode ser encerrado. Encerrado ou cancelado → `409 bilhete_ja_encerrado`.                | UC2 |
| RN-13 | Só bilhete `aberto` pode ser cancelado. Caso contrário → `409 bilhete_nao_aberto`.                          | UC5 |
| RN-14 | Cancelamento não gera `saida`, `minutos` nem `valor_centavos`.                                              | UC5 |
| RN-15 | Bilhete inexistente (ou `id` inválido) → `404 bilhete_nao_encontrado`.                                      | UC2, UC5 |
| RN-16 | Ordenação "mais recentes primeiro" = `entrada` decrescente, desempate por `id` decrescente.                 | UC3, UC6 |
| RN-17 | Placa sem histórico retorna array vazio com `200`.                                                          | UC6 |
| RN-18 | Relatório considera apenas bilhetes `encerrado` com `saida` na data informada (fuso -03:00).                | UC4 |
| RN-19 | `tempo_medio_minutos` = média de `minutos` dos encerrados no dia, arredondando 0,5 para cima; sem bilhetes → `0`. | UC4 |
| RN-20 | `data` deve estar no formato `AAAA-MM-DD` e ser uma data existente. Ausente ou inválida → `422 data_invalida`. | UC4 |

## 5. Fora de Escopo

- Back-office, painel administrativo ou qualquer front-end.
- Autenticação, autorização e controle de usuários.
- Pagamento, emissão de nota fiscal ou integração com meios de pagamento.
- Persistência durável (banco é em memória; dados se perdem ao reiniciar).
- Controle de capacidade/número de vagas físicas do estacionamento.
- Tarifas diferenciadas por tipo de veículo, dia da semana ou horário.
- Cobrança acumulada de bilhetes que atravessam mais de um dia (o teto é por bilhete).
- Edição ou exclusão de bilhetes.
- Reabertura de bilhetes encerrados ou cancelados.


## 6. Critérios de Aceite

### UC1 — Abrir bilhete
- **CA-01.1** Dado body `{"placa": "ABC1D23"}`, quando `POST /bilhetes`, então `201` com `id`, `placa: "ABC1D23"`, `entrada` com offset `-03:00` e `status: "aberto"`.
- **CA-01.2** Dado body com `entrada` ISO-8601 válida, quando `POST /bilhetes`, então `entrada` retornada corresponde ao mesmo instante informado, em `-03:00`.
- **CA-01.3** Dado body sem `placa`, com placa minúscula, com 6 ou 8 caracteres ou com caractere especial, então `422 {"erro": "placa_invalida"}`.
- **CA-01.4** Dado body com `entrada` inválida (ex.: `"ontem"`, `"2026-10-07 10:00"`, sem fuso), então `422 {"erro": "entrada_invalida"}`.

### UC2 — Encerrar bilhete
- **CA-02.1** Dado bilhete aberto, quando `POST /bilhetes/{id}/encerramento`, então `200` com `id`, `placa`, `entrada`, `saida`, `minutos` e `valor_centavos` inteiro.
- **CA-02.2** Dado bilhete com duração igual a exatamente N frações (acima da tolerância), então `valor_centavos = N × valorFracao`.
- **CA-02.3** Dado bilhete com duração de N frações + 1 minuto, então `valor_centavos = (N + 1) × valorFracao`.
- **CA-02.4** Dado bilhete cuja cobrança ultrapassaria o teto, então `valor_centavos = TETO_DIARIO_CENTAVOS`.
- **CA-02.5** Dado bilhete já encerrado ou cancelado, então `409 {"erro": "bilhete_ja_encerrado"}`.
- **CA-02.6** Dado `id` inexistente, então `404 {"erro": "bilhete_nao_encontrado"}`.
- **CA-02.7** Nenhum campo monetário da resposta contém ponto decimal.

### UC3 — Listar ativos
- **CA-03.1** Dados bilhetes abertos, encerrados e cancelados, quando `GET /bilhetes/ativos`, então `200` contendo apenas os `aberto`.
- **CA-03.2** O array está ordenado do bilhete com `entrada` mais recente para o mais antigo.
- **CA-03.3** Sem bilhetes abertos, retorna `200` com `[]`.

### UC4 — Relatório diário
- **CA-04.1** Dados bilhetes encerrados na data, quando `GET /relatorios/diario?data=AAAA-MM-DD`, então `200` com `data` igual à informada, `total_bilhetes`, `faturamento_centavos` (soma dos valores) e `tempo_medio_minutos` inteiro.
- **CA-04.2** Bilhetes abertos, cancelados ou encerrados em outra data não entram no relatório.
- **CA-04.3** Média com parte decimal exatamente 0,5 arredonda para cima (ex.: 10 e 11 minutos → `11`).
- **CA-04.4** Data sem bilhetes encerrados → `200` com `total_bilhetes: 0`, `faturamento_centavos: 0`, `tempo_medio_minutos: 0`.
- **CA-04.5** `data` ausente, em formato diferente (ex.: `07/10/2026`) ou inexistente (ex.: `2026-02-30`) → `422 {"erro": "data_invalida"}`.

### UC5 — Cancelar bilhete
- **CA-05.1** Dado bilhete aberto, quando `POST /bilhetes/{id}/cancelamento`, então `200` com `status: "cancelado"`, sem `saida` e sem `valor_centavos`.
- **CA-05.2** Dado bilhete encerrado ou já cancelado, então `409 {"erro": "bilhete_nao_aberto"}`.
- **CA-05.3** Dado `id` inexistente, então `404 {"erro": "bilhete_nao_encontrado"}`.
- **CA-05.4** Bilhete cancelado não aparece em `GET /bilhetes/ativos` nem no relatório diário.

### UC6 — Histórico por placa
- **CA-06.1** Dada placa com bilhetes em vários status, quando `GET /bilhetes?placa=ABC1D23`, então `200` com todos eles, mais recentes primeiro.
- **CA-06.2** Bilhetes de outras placas não aparecem.
- **CA-06.3** Placa que nunca estacionou → `200` com `[]`.
- **CA-06.4** Placa inválida no parâmetro → `422 {"erro": "placa_invalida"}`.

### UC7 — Tolerância gratuita
- **CA-07.1** Duração de 0 minutos até `TOLERANCIA_MINUTOS` (inclusive) → `valor_centavos: 0`.
- **CA-07.2** Duração de `TOLERANCIA_MINUTOS + 1` → cobra a partir do primeiro minuto (sem descontar a tolerância).
- **CA-07.3** Com `TOLERANCIA_MINUTOS = 0`, qualquer duração ≥ 1 minuto é cobrada.

### UC8 — Uma vaga por placa
- **CA-08.1** Dada placa com bilhete aberto, quando `POST /bilhetes` com a mesma placa, então `409 {"erro": "bilhete_em_aberto"}`.
- **CA-08.2** Após encerrar o bilhete, nova abertura para a placa retorna `201`.
- **CA-08.3** Após cancelar o bilhete, nova abertura para a placa retorna `201`.



