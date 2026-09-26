# T1 — Otimização de consultas no Supabase

Registro da execução da atividade de Infraestrutura para Gestão de Dados, utilizando PostgreSQL no Supabase e o dataset de transporte aéreo disponibilizado para o trabalho.

O trabalho contempla a preparação da base, sete consultas, análise dos planos de execução, criação experimental de índices, reescritas e comparação dos resultados. As instruções foram obtidas do enunciado `Trabalho_Otimizacao_Consultas.pdf`. Os PDFs não fazem parte deste repositório; os comandos e as evidências necessários à leitura da atividade estão neste Markdown.

## Conteúdo do repositório

```text
README.md                 # procedimento, SQL, resultados e planos
dados/TAB_AIR_*.tsv        # dez arquivos originais do dataset
```

O dataset foi mantido localmente. Sua publicação está pendente da confirmação de que os registros de passageiros são fictícios. Para reproduzir a carga, será necessário obter os dez TSVs e colocá-los na pasta `dados/`.

## Ambiente e decisões

- Banco: PostgreSQL 17.6, hospedado no Supabase.
- Schema utilizado: `t1_air`, separado das tabelas de outras aplicações.
- Execução e carga: acesso administrativo pelo MCP do Supabase.
- O esquema originalmente destinado ao Oracle foi adaptado para PostgreSQL. `NUMBER` foi convertido para inteiros ou `numeric`, conforme a coluna; datas com horário foram preservadas como `timestamp without time zone`.
- A base inicial foi criada sem chaves primárias, estrangeiras ou índices nas dez tabelas de dados, seguindo a estrutura de origem e permitindo observar os planos antes da otimização. A tabela técnica `import_batches` possui uma chave primária para controlar os lotes.
- O schema recebeu `REVOKE ALL ... FROM PUBLIC` e as tabelas receberam RLS sem políticas de acesso público. A atividade foi executada com acesso administrativo.
- Os custos apresentados são do PostgreSQL. Não representam milissegundos e não devem ser comparados numericamente com custos do Oracle.

## 1. Preparação e validação do dataset

Os arquivos originais foram mantidos sem alteração. A leitura foi feita com um parser de CSV configurado para tabulação, porque há campos entre aspas e até quebras de linha dentro de campos. Contar linhas físicas do arquivo pode produzir uma contagem incorreta de registros.

Formato observado: UTF-8, separador de tabulação, sem cabeçalho; campos textuais podem estar entre aspas. Valores vazios foram convertidos para `NULL`; vírgulas decimais para ponto; datas `dd/mm/aa` ou `dd/mm/aa HH:MM` para o formato ISO. Não foram incluídas credenciais de acesso no repositório.

### Quantidade de registros

| Arquivo / tabela | Registros |
|---|---:|
| TAB_AIR_AIRLINES | 113 |
| TAB_AIR_AIRPLANE_TYPES | 342 |
| TAB_AIR_AIRPLANES | 5,583 |
| TAB_AIR_AIRPORTS | 9,854 |
| TAB_AIR_AIRPORTS_GEO | 9,854 |
| TAB_AIR_BOOKINGS | 122,244 |
| TAB_AIR_FLIGHTS | 1,498 |
| TAB_AIR_FLIGHTS_SCHEDULES | 1,424 |
| TAB_AIR_PASSENGERS | 36,095 |
| TAB_AIR_PASSENGERS_DETAILS | 36,095 |
| **Total** | **223,102** |

### Mapeamento essencial das reservas

O TSV `TAB_AIR_BOOKINGS.tsv` tem as colunas nesta ordem:

```text
BOOKING_ID, FLIGHT_ID, PASSENGER_ID, SEAT, PRICE
```

A tabela foi definida nesta ordem:

```text
BOOKING_ID, PASSENGER_ID, FLIGHT_ID, SEAT, PRICE
```

Na primeira carga, os dois identificadores foram interpretados na ordem da tabela, o que impediu as junções e fez as primeiras consultas retornarem zero linhas. Identificamos o problema comparando os IDs das tabelas e testando a associação com as colunas invertidas. Corrigimos os valores já carregados uma única vez e ajustamos o conversor para trocar as posições 2 e 3 antes de importar reservas.

**A base atual já está corrigida.** Para reproduzir, use o conversor abaixo, que faz o mapeamento correto desde a carga. Os TSVs mantêm a ordem original.

### Estrutura utilizada

Execute uma vez em um banco destinado à atividade, antes de carregar os dados. A criação falhará se o schema já existir; não é necessário recriar a base atualmente preparada.

<details>
<summary>SQL completo de criação do schema e das tabelas</summary>

```sql
-- T1: transcricao das tabelas Oracle para PostgreSQL/Supabase.
-- Schema privado para nao misturar com as tabelas da aplicacao.
-- Sem indices ou chaves inicialmente: os planos originais devem refletir os dados brutos.

CREATE SCHEMA t1_air;
REVOKE ALL ON SCHEMA t1_air FROM PUBLIC;

CREATE TABLE t1_air.tab_air_airlines (
    airline_id integer NOT NULL,
    iata char(2) NOT NULL,
    airline_name varchar(30) NOT NULL,
    base_airport_id integer NOT NULL
);

CREATE TABLE t1_air.tab_air_airplane_types (
    airplane_type_id integer NOT NULL,
    name varchar(50) NOT NULL
);

CREATE TABLE t1_air.tab_air_airplanes (
    airplane_id integer NOT NULL,
    airline_id integer NOT NULL,
    airplane_type_id integer NOT NULL,
    capacity integer NOT NULL
);

CREATE TABLE t1_air.tab_air_airports (
    airport_id integer NOT NULL,
    iata char(3),
    icao char(4) NOT NULL,
    name varchar(30) NOT NULL
);

CREATE TABLE t1_air.tab_air_airports_geo (
    airport_id integer NOT NULL,
    name varchar(50) NOT NULL,
    city varchar(50),
    country varchar(50) NOT NULL,
    latitude numeric(11,8) NOT NULL,
    longitude numeric(11,8) NOT NULL
);

CREATE TABLE t1_air.tab_air_flights_schedules (
    flightno char(8) NOT NULL,
    airline_id integer NOT NULL,
    from_airport_id integer NOT NULL,
    to_airport_id integer NOT NULL,
    departure timestamp without time zone NOT NULL,
    arrival timestamp without time zone NOT NULL,
    monday integer NOT NULL,
    tuesday integer NOT NULL,
    wednesday integer NOT NULL,
    thursday integer NOT NULL,
    friday integer NOT NULL,
    saturday integer NOT NULL,
    sunday integer NOT NULL
);

CREATE TABLE t1_air.tab_air_flights (
    flight_id bigint NOT NULL,
    flightno char(8) NOT NULL,
    airline_id integer NOT NULL,
    from_airport_id integer NOT NULL,
    to_airport_id integer NOT NULL,
    airplane_id integer NOT NULL,
    departure timestamp without time zone NOT NULL,
    arrival timestamp without time zone NOT NULL
);

CREATE TABLE t1_air.tab_air_passengers (
    passenger_id bigint NOT NULL,
    passportno char(9) NOT NULL,
    firstname varchar(100) NOT NULL,
    lastname varchar(100) NOT NULL
);

CREATE TABLE t1_air.tab_air_passengers_details (
    passenger_id bigint NOT NULL,
    birthdate timestamp without time zone NOT NULL,
    sex char(1) NOT NULL,
    street varchar(100),
    city varchar(100) NOT NULL,
    zip integer NOT NULL,
    country varchar(50),
    emailaddress varchar(120),
    telephoneno varchar(30)
);

CREATE TABLE t1_air.tab_air_bookings (
    booking_id bigint NOT NULL,
    passenger_id bigint NOT NULL,
    flight_id bigint NOT NULL,
    seat char(4),
    price numeric(10,2) NOT NULL
);

-- Controle de carga para poder retomar sem duplicar linhas.
CREATE TABLE t1_air.import_batches (
    source_file text NOT NULL,
    batch_no integer NOT NULL,
    row_count integer NOT NULL,
    imported_at timestamp with time zone NOT NULL DEFAULT now(),
    PRIMARY KEY (source_file, batch_no)
);

ALTER TABLE t1_air.tab_air_airlines ENABLE ROW LEVEL SECURITY;
ALTER TABLE t1_air.tab_air_airplane_types ENABLE ROW LEVEL SECURITY;
ALTER TABLE t1_air.tab_air_airplanes ENABLE ROW LEVEL SECURITY;
ALTER TABLE t1_air.tab_air_airports ENABLE ROW LEVEL SECURITY;
ALTER TABLE t1_air.tab_air_airports_geo ENABLE ROW LEVEL SECURITY;
ALTER TABLE t1_air.tab_air_flights_schedules ENABLE ROW LEVEL SECURITY;
ALTER TABLE t1_air.tab_air_flights ENABLE ROW LEVEL SECURITY;
ALTER TABLE t1_air.tab_air_passengers ENABLE ROW LEVEL SECURITY;
ALTER TABLE t1_air.tab_air_passengers_details ENABLE ROW LEVEL SECURITY;
ALTER TABLE t1_air.tab_air_bookings ENABLE ROW LEVEL SECURITY;
ALTER TABLE t1_air.import_batches ENABLE ROW LEVEL SECURITY;
```

</details>

### Conversão e carga em lotes

Para reproduzir a conversão, salve o código a seguir como `preparar_carga.py` na raiz do clone, ao lado deste README, e execute `python preparar_carga.py` (Python 3.10 ou superior, somente biblioteca padrão). Ele gera SQL em `tmp/t1_air_import/` e um manifesto com as contagens. Esse script temporário não integra os arquivos publicados.

Cada lote executa a inserção e seu registro em `import_batches` no mesmo bloco transacional. Isso permite retomar lotes já concluídos sem duplicar a carga, desde que arquivos, conversor e limites dos lotes permaneçam iguais. O procedimento pressupõe uma execução de carga por vez. Se um lote exceder o limite do transporte, divida-o mantendo identificadores de lote distintos.

Execute os arquivos SQL gerados, em ordem por tabela e lote, usando o SQL Editor ou uma conexão administrativa ao PostgreSQL. Na atividade, esses comandos foram enviados pelo MCP. A tabela técnica fica no mesmo schema privado.

<details>
<summary>Conversor de TSV para lotes SQL</summary>

```python
"""Convert the supplied Oracle TSV exports into resumable PostgreSQL batches.

Run from the repository root:
  python dados/prepare_supabase_import.py

The generated files under tmp/t1_air_import are scratch files. The source TSVs
and the schema definition in dados/supabase_schema.sql are the durable inputs.
"""

from __future__ import annotations

import csv
import json
import sys
from datetime import datetime
from pathlib import Path


ROOT = Path(__file__).resolve().parent
SOURCE = ROOT / "dados"
OUTPUT = ROOT / "tmp" / "t1_air_import"
MAX_BATCH_BYTES = 110_000

# Column kinds: i = integer, n = decimal, d = Oracle DATE, s = string.
TABLES = {
    "TAB_AIR_AIRLINES": "issi",
    "TAB_AIR_AIRPLANE_TYPES": "is",
    "TAB_AIR_AIRPLANES": "iiii",
    "TAB_AIR_AIRPORTS": "isss",
    "TAB_AIR_AIRPORTS_GEO": "isssnn",
    "TAB_AIR_BOOKINGS": "iiisn",
    "TAB_AIR_FLIGHTS": "isiiiidd",
    "TAB_AIR_FLIGHTS_SCHEDULES": "siiiddiiiiiii",
    "TAB_AIR_PASSENGERS": "isss",
    "TAB_AIR_PASSENGERS_DETAILS": "idsssisss",
}


def ordered_fields(stem: str, fields: list[str]) -> list[str]:
    # The bookings export is BOOKING_ID, FLIGHT_ID, PASSENGER_ID, SEAT, PRICE.
    # The table definition is BOOKING_ID, PASSENGER_ID, FLIGHT_ID, SEAT, PRICE.
    if stem == "TAB_AIR_BOOKINGS":
        return [fields[0], fields[2], fields[1], fields[3], fields[4]]
    return fields


def literal(value: str, kind: str, source: Path, row_no: int) -> str:
    if value == "":
        return "NULL"
    if "\x00" in value:
        raise ValueError(f"NUL byte in {source.name}, row {row_no}")
    if kind == "s":
        return "'" + value.replace("'", "''") + "'"
    if kind == "i":
        return str(int(value))
    if kind == "n":
        return str(float(value.replace(",", "."))) if "e" in value.lower() else value.replace(",", ".")
    if kind == "d":
        pattern = "%d/%m/%y %H:%M" if " " in value else "%d/%m/%y"
        parsed = datetime.strptime(value, pattern)
        return "'" + parsed.strftime("%Y-%m-%d %H:%M:%S") + "'"
    raise ValueError(f"Unknown column kind: {kind}")


def write_batch(stem: str, batch_no: int, rows: list[str], source_name: str) -> str:
    table = stem.lower()
    values = ",\n".join(rows)
    sql = (
        "DO $load$\nBEGIN\n"
        "  IF NOT EXISTS (SELECT 1 FROM t1_air.import_batches "
        f"WHERE source_file = '{source_name}' AND batch_no = {batch_no}) THEN\n"
        f"    INSERT INTO t1_air.{table} VALUES\n{values};\n"
        "    INSERT INTO t1_air.import_batches (source_file, batch_no, row_count) "
        f"VALUES ('{source_name}', {batch_no}, {len(rows)});\n"
        "  END IF;\nEND\n$load$;\n"
    )
    path = OUTPUT / f"{stem.lower()}_{batch_no:04d}.sql"
    path.write_text(sql, encoding="utf-8")
    return path.name


def main() -> None:
    OUTPUT.mkdir(parents=True, exist_ok=True)
    manifest = []
    for stem, kinds in TABLES.items():
        source = SOURCE / f"{stem}.tsv"
        batch_no = 0
        rows: list[str] = []
        batch_bytes = 0
        total = 0
        files = []
        with source.open(encoding="utf-8-sig", newline="") as stream:
            for row_no, fields in enumerate(csv.reader(stream, delimiter="\t"), start=1):
                if len(fields) != len(kinds):
                    raise ValueError(
                        f"{source.name}, row {row_no}: {len(fields)} fields; expected {len(kinds)}"
                    )
                fields = ordered_fields(stem, fields)
                tuple_sql = "      (" + ", ".join(
                    literal(value, kind, source, row_no)
                    for value, kind in zip(fields, kinds)
                ) + ")"
                size = len(tuple_sql.encode("utf-8")) + 2
                if rows and batch_bytes + size > MAX_BATCH_BYTES:
                    files.append(write_batch(stem, batch_no, rows, source.name))
                    batch_no += 1
                    rows = []
                    batch_bytes = 0
                rows.append(tuple_sql)
                batch_bytes += size
                total += 1
        if rows:
            files.append(write_batch(stem, batch_no, rows, source.name))
        manifest.append({"table": stem.lower(), "source": source.name, "rows": total, "batches": files})
    (OUTPUT / "manifest.json").write_text(
        json.dumps(manifest, ensure_ascii=False, indent=2), encoding="utf-8"
    )
    for item in manifest:
        print(f"{item['source']}: {item['rows']} rows in {len(item['batches'])} batches")


if __name__ == "__main__":
    main()
```

</details>

Após a carga, conferimos as contagens das dez tabelas e atualizamos as estatísticas com `ANALYZE`. Para repetir a atualização:

```sql
ANALYZE t1_air.tab_air_airlines;
ANALYZE t1_air.tab_air_airplane_types;
ANALYZE t1_air.tab_air_airplanes;
ANALYZE t1_air.tab_air_airports;
ANALYZE t1_air.tab_air_airports_geo;
ANALYZE t1_air.tab_air_bookings;
ANALYZE t1_air.tab_air_flights;
ANALYZE t1_air.tab_air_flights_schedules;
ANALYZE t1_air.tab_air_passengers;
ANALYZE t1_air.tab_air_passengers_details;
```

## 2. Método de análise

1. Coletamos o plano de cada consulta original antes de criar índices de otimização.
2. Registramos o plano com `EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)`, que executa a consulta e mostra estimativas, linhas efetivas, métodos de acesso, junções, ordenação e buffers.
3. Criamos os índices propostos para uma consulta por vez. Não desativamos métodos do planejador para forçar o uso de índices.
4. Medimos a consulta original com esses índices. Nas questões 4, 5 e 6, também testamos uma reescrita.
5. Conferimos as quantidades. Para as reescritas, usamos `EXCEPT ALL` nos dois sentidos, preservando também a multiplicidade das linhas: todas as comparações tiveram zero diferenças.
6. Removemos os índices experimentais ao terminar cada questão, evitando interferência nas questões seguintes. A base permaneceu carregada, sem os índices de otimização.

Para executar um plano, coloque o comando abaixo imediatamente antes do `SELECT` desejado:

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
-- SELECT ...
```

### Limitações das medições

Os tempos correspondem às execuções instrumentadas registradas, sem séries repetidas ou controle de cache e concorrência. São evidências dessa execução, não garantias de desempenho futuro. `Planning Time` e `Execution Time` são medidas distintas; a tabela compara somente `Execution Time`. Estatísticas, estado do cache e configuração podem mudar o plano ao reproduzir.

## 3. Resumo dos resultados

| Questão | Linhas | Custo original | Custo final | Execução original (ms) | Execução final (ms) | Reescrita |
|---|---:|---:|---:|---:|---:|---|
| Q1 | 5 | 4031.88 | 17.45 | 16.384 | 0.577 | Não |
| Q2 | 248 | 4542.87 | 165.70 | 34.767 | 7.036 | Não |
| Q3 | 37 | 5039.71 | 58.16 | 33.926 | 8.089 | Não |
| Q4 | 99 | 14488.08 | 6842.50 | 209.120 | 128.314 | Sim |
| Q5 | 27 | 4687.34 | 280.57 | 35.200 | 2.972 | Sim |
| Q6 | 7382 | 1386.04 | 1270.98 | 35.683 | 36.875 | Sim |
| Q7 | 268 | 1936.97 | 649.89 | 32.065 | 4.596 | Não |

Q1 a Q6 coincidiram com as quantidades esperadas no enunciado. Q7 retornou 268 linhas; o enunciado não informou uma quantidade esperada. **Q6 reduziu o custo estimado, mas não melhorou o tempo observado**: passou de 35,683 para 36,875 ms.

## 4. Consultas, estratégias e evidências

Em cada questão, execute primeiro a consulta original e seu plano sem os índices experimentais. Em seguida, crie os índices apresentados, execute o plano com índices e, quando houver, o plano da reescrita. O comando de remoção encerra o experimento. Para comparar todas as bases originais antes dos testes, colete primeiro os sete planos sem índices.

### Q1 — Reservas do passageiro 17638 em 2023

**Resultado:** 5 linhas. Custo 4031.88 → 17.45; execução 16.384 → 0.577 ms.

#### Consulta original

```sql
SELECT b.booking_id, b.passenger_id, b.seat, b.price,
       f.flight_id, f.flightno, f.departure, f.arrival,
       a.airline_id, a.airline_name,
       ao.iata AS origin_iata, ao.name AS origin_name,
       ad.iata AS destination_iata, ad.name AS destination_name
FROM t1_air.tab_air_bookings b
JOIN t1_air.tab_air_flights f ON f.flight_id = b.flight_id
JOIN t1_air.tab_air_airlines a ON a.airline_id = f.airline_id
JOIN t1_air.tab_air_airports ao ON ao.airport_id = f.from_airport_id
JOIN t1_air.tab_air_airports ad ON ad.airport_id = f.to_airport_id
WHERE b.passenger_id = 17638
  AND f.departure >= TIMESTAMP '2023-01-01'
  AND f.departure < TIMESTAMP '2024-01-01'
ORDER BY f.departure;
```

#### Análise do plano original

O plano estima uma linha na saída, mas retorna cinco. A leitura sequencial de reservas examina 122.244 registros e elimina 122.230 pelo filtro: sobram 14 reservas do passageiro. A leitura dos voos seleciona 348 partidas em 2023; a junção reduz o conjunto a cinco linhas.

As duas junções com aeroportos leem 9.854 linhas cada por Seq Scan e usam Hash Join. Há também Nested Loop com companhia aérea e uma ordenação quicksort de apenas 25 kB. O principal alvo é o acesso seletivo às reservas e às tabelas consultadas por identificador, e não a pequena ordenação final.

#### Estratégia adotada

Criar um índice composto em reservas, iniciando pelo passageiro e seguido pelo voo, além de índices nos identificadores de voo, companhia e aeroporto. A primeira coluna atende ao filtro de igualdade e a segunda disponibiliza a chave da junção. O mesmo índice de aeroportos atende aos dois aliases, origem e destino.

A consulta permanece inalterada. Os quatro índices são utilizados no plano escolhido. O intervalo de datas usa início inclusivo e fim exclusivo, incluindo todas as horas de 31/12/2023.

```sql
CREATE INDEX t1_q1_booking_passenger ON t1_air.tab_air_bookings (passenger_id, flight_id);
CREATE INDEX t1_q1_flight_id ON t1_air.tab_air_flights (flight_id);
CREATE INDEX t1_q1_airline_id ON t1_air.tab_air_airlines (airline_id);
CREATE INDEX t1_q1_airport_id ON t1_air.tab_air_airports (airport_id);
```

#### Planos registrados

<details>
<summary>Plano original — sem índices de otimização</summary>

```text
Sort  (cost=4031.87..4031.88 rows=1 width=111) (actual time=16.206..16.211 rows=5 loops=1)
  Sort Key: f.departure
  Sort Method: quicksort  Memory: 25kB
  Buffers: shared hit=2194
  ->  Nested Loop  (cost=3826.82..4031.86 rows=1 width=111) (actual time=14.727..16.176 rows=5 loops=1)
        Join Filter: (f.airline_id = a.airline_id)
        Rows Removed by Join Filter: 560
        Buffers: shared hit=2191
        ->  Hash Join  (cost=3826.82..4028.32 rows=1 width=94) (actual time=14.677..16.045 rows=5 loops=1)
              Hash Cond: (ad.airport_id = f.to_airport_id)
              Buffers: shared hit=2186
              ->  Seq Scan on tab_air_airports ad  (cost=0.00..164.54 rows=9854 width=19) (actual time=0.014..0.721 rows=9854 loops=1)
                    Buffers: shared hit=66
              ->  Hash  (cost=3826.80..3826.80 rows=1 width=83) (actual time=14.246..14.249 rows=5 loops=1)
                    Buckets: 1024  Batches: 1  Memory Usage: 9kB
                    Buffers: shared hit=2120
                    ->  Hash Join  (cost=3625.30..3826.80 rows=1 width=83) (actual time=12.625..14.240 rows=5 loops=1)
                          Hash Cond: (ao.airport_id = f.from_airport_id)
                          Buffers: shared hit=2120
                          ->  Seq Scan on tab_air_airports ao  (cost=0.00..164.54 rows=9854 width=19) (actual time=0.009..0.814 rows=9854 loops=1)
                                Buffers: shared hit=66
                          ->  Hash  (cost=3625.29..3625.29 rows=1 width=72) (actual time=12.333..12.334 rows=5 loops=1)
                                Buckets: 1024  Batches: 1  Memory Usage: 9kB
                                Buffers: shared hit=2054
                                ->  Nested Loop  (cost=0.00..3625.29 rows=1 width=72) (actual time=11.434..12.314 rows=5 loops=1)
                                      Join Filter: (b.flight_id = f.flight_id)
                                      Rows Removed by Join Filter: 4867
                                      Buffers: shared hit=2054
                                      ->  Seq Scan on tab_air_flights f  (cost=0.00..38.47 rows=346 width=45) (actual time=0.017..0.199 rows=348 loops=1)
                                            Filter: ((departure >= '2023-01-01 00:00:00'::timestamp without time zone) AND (departure < '2024-01-01 00:00:00'::timestamp without time zone))
                                            Rows Removed by Filter: 1150
                                            Buffers: shared hit=16
                                      ->  Materialize  (cost=0.00..3566.07 rows=4 width=35) (actual time=0.007..0.033 rows=14 loops=348)
                                            Buffers: shared hit=2038
                                            ->  Seq Scan on tab_air_bookings b  (cost=0.00..3566.05 rows=4 width=35) (actual time=2.380..11.254 rows=14 loops=1)
                                                  Filter: (passenger_id = 17638)
                                                  Rows Removed by Filter: 122230
                                                  Buffers: shared hit=2038
        ->  Seq Scan on tab_air_airlines a  (cost=0.00..2.13 rows=113 width=21) (actual time=0.008..0.015 rows=113 loops=5)
              Buffers: shared hit=5
Planning:
  Buffers: shared hit=330
Planning Time: 2.277 ms
Execution Time: 16.384 ms
```

</details>

<details>
<summary>Plano final — consulta escolhida com índices</summary>

```text
Sort  (cost=17.45..17.45 rows=1 width=111) (actual time=0.384..0.386 rows=5 loops=1)
  Sort Key: f.departure
  Sort Method: quicksort  Memory: 25kB
  Buffers: shared hit=91 read=14
  ->  Nested Loop  (cost=1.41..17.44 rows=1 width=111) (actual time=0.120..0.354 rows=5 loops=1)
        Buffers: shared hit=88 read=14
        ->  Nested Loop  (cost=1.12..16.82 rows=1 width=100) (actual time=0.100..0.289 rows=5 loops=1)
              Buffers: shared hit=77 read=10
              ->  Nested Loop  (cost=0.84..16.21 rows=1 width=89) (actual time=0.079..0.228 rows=5 loops=1)
                    Buffers: shared hit=66 read=6
                    ->  Nested Loop  (cost=0.70..16.03 rows=1 width=72) (actual time=0.068..0.209 rows=5 loops=1)
                          Buffers: shared hit=56 read=6
                          ->  Index Scan using t1_q1_booking_passenger on tab_air_bookings b  (cost=0.42..5.99 rows=4 width=35) (actual time=0.053..0.121 rows=14 loops=1)
                                Index Cond: (passenger_id = 17638)
                                Buffers: shared hit=17 read=3
                          ->  Index Scan using t1_q1_flight_id on tab_air_flights f  (cost=0.28..2.50 rows=1 width=45) (actual time=0.005..0.006 rows=0 loops=14)
                                Index Cond: (flight_id = b.flight_id)
                                Filter: ((departure >= '2023-01-01 00:00:00'::timestamp without time zone) AND (departure < '2024-01-01 00:00:00'::timestamp without time zone))
                                Rows Removed by Filter: 1
                                Buffers: shared hit=39 read=3
                    ->  Index Scan using t1_q1_airline_id on tab_air_airlines a  (cost=0.14..0.17 rows=1 width=21) (actual time=0.003..0.003 rows=1 loops=5)
                          Index Cond: (airline_id = f.airline_id)
                          Buffers: shared hit=10
              ->  Index Scan using t1_q1_airport_id on tab_air_airports ao  (cost=0.29..0.60 rows=1 width=19) (actual time=0.011..0.011 rows=1 loops=5)
                    Index Cond: (airport_id = f.from_airport_id)
                    Buffers: shared hit=11 read=4
        ->  Index Scan using t1_q1_airport_id on tab_air_airports ad  (cost=0.29..0.60 rows=1 width=19) (actual time=0.012..0.012 rows=1 loops=5)
              Index Cond: (airport_id = f.to_airport_id)
              Buffers: shared hit=11 read=4
Planning:
  Buffers: shared hit=403 read=11
Planning Time: 10.135 ms
Execution Time: 0.577 ms
```

</details>

#### Conclusão

O plano troca as leituras amplas por Index Scan e Nested Loop com buscas pontuais. O filtro de passageiro recupera 14 reservas pelo índice, e a data deixa cinco voos. Os cinco resultados são preservados. A ordenação final continua pequena; a melhoria decorre sobretudo da seletividade e dos acessos por chave.

#### Encerramento do experimento

```sql
DROP INDEX IF EXISTS t1_air.t1_q1_booking_passenger;
DROP INDEX IF EXISTS t1_air.t1_q1_flight_id;
DROP INDEX IF EXISTS t1_air.t1_q1_airline_id;
DROP INDEX IF EXISTS t1_air.t1_q1_airport_id;
```

### Q2 — Passageiros da companhia 47 no primeiro trimestre de 2023

**Resultado:** 248 linhas. Custo 4542.87 → 165.70; execução 34.767 → 7.036 ms.

#### Consulta original

```sql
SELECT p.passenger_id, p.passportno, p.firstname, p.lastname,
       b.booking_id, b.seat, b.price,
       f.flight_id, f.flightno, f.airline_id,
       f.from_airport_id, f.to_airport_id, f.departure, f.arrival
FROM t1_air.tab_air_passengers p
JOIN t1_air.tab_air_bookings b ON b.passenger_id = p.passenger_id
JOIN t1_air.tab_air_flights f ON f.flight_id = b.flight_id
WHERE f.airline_id = 47
  AND f.departure >= TIMESTAMP '2023-01-01'
  AND f.departure < TIMESTAMP '2023-04-01'
ORDER BY f.departure, p.firstname, p.lastname;
```

#### Análise do plano original

O filtro de companhia e datas estima um voo e encontra dois. A primeira Hash Join associa as reservas a esses voos, produzindo 248 linhas frente a 82 estimadas. Mesmo para esse pequeno conjunto, o plano original lê todas as 122.244 reservas e todos os 36.095 passageiros.

A ordenação final por partida, nome e sobrenome usa quicksort e 54 kB. O problema central é a ausência de caminhos de acesso para o filtro composto em voos e para as junções por voo e passageiro. A estimativa de 82 linhas na raiz subestima as 248 efetivas.

#### Estratégia adotada

Usar índice em voos com companhia primeiro, data em seguida e identificador do voo ao final. Criar também índice composto de reservas por voo e passageiro, e índice de passageiros pelo identificador. O filtro de companhia é de igualdade; a data é um intervalo.

A consulta não é reescrita. O plano usa Index Scan para os dois voos, Bitmap Index Scan/Bitmap Heap Scan para suas reservas e Index Scan para os passageiros. O índice dos voos fornece ordenação parcial por partida, aproveitada por Incremental Sort.

```sql
CREATE INDEX t1_q2_flight_airline_date ON t1_air.tab_air_flights (airline_id, departure, flight_id);
CREATE INDEX t1_q2_booking_flight ON t1_air.tab_air_bookings (flight_id, passenger_id);
CREATE INDEX t1_q2_passenger_id ON t1_air.tab_air_passengers (passenger_id);
```

#### Planos registrados

<details>
<summary>Plano original — sem índices de otimização</summary>

```text
Sort  (cost=4542.66..4542.87 rows=82 width=95) (actual time=34.604..34.621 rows=248 loops=1)
  Sort Key: f.departure, p.firstname, p.lastname
  Sort Method: quicksort  Memory: 54kB
  Buffers: shared hit=2340
  ->  Hash Join  (cost=3762.93..4540.05 rows=82 width=95) (actual time=27.072..34.107 rows=248 loops=1)
        Hash Cond: (p.passenger_id = b.passenger_id)
        Buffers: shared hit=2334
        ->  Seq Scan on tab_air_passengers p  (cost=0.00..640.95 rows=36095 width=31) (actual time=0.014..3.064 rows=36095 loops=1)
              Buffers: shared hit=280
        ->  Hash  (cost=3761.90..3761.90 rows=82 width=72) (actual time=26.849..26.851 rows=248 loops=1)
              Buckets: 1024  Batches: 1  Memory Usage: 36kB
              Buffers: shared hit=2054
              ->  Hash Join  (cost=42.23..3761.90 rows=82 width=72) (actual time=3.643..26.759 rows=248 loops=1)
                    Hash Cond: (b.flight_id = f.flight_id)
                    Buffers: shared hit=2054
                    ->  Seq Scan on tab_air_bookings b  (cost=0.00..3260.44 rows=122244 width=35) (actual time=1.270..12.969 rows=122244 loops=1)
                          Buffers: shared hit=2038
                    ->  Hash  (cost=42.22..42.22 rows=1 width=45) (actual time=0.222..0.223 rows=2 loops=1)
                          Buckets: 1024  Batches: 1  Memory Usage: 9kB
                          Buffers: shared hit=16
                          ->  Seq Scan on tab_air_flights f  (cost=0.00..42.22 rows=1 width=45) (actual time=0.050..0.192 rows=2 loops=1)
                                Filter: ((departure >= '2023-01-01 00:00:00'::timestamp without time zone) AND (departure < '2023-04-01 00:00:00'::timestamp without time zone) AND (airline_id = 47))
                                Rows Removed by Filter: 1496
                                Buffers: shared hit=16
Planning:
  Buffers: shared hit=245
Planning Time: 0.864 ms
Execution Time: 34.767 ms
```

</details>

<details>
<summary>Plano final — consulta escolhida com índices</summary>

```text
Incremental Sort  (cost=164.66..165.70 rows=82 width=95) (actual time=6.300..6.895 rows=248 loops=1)
  Sort Key: f.departure, p.firstname, p.lastname
  Presorted Key: f.departure
  Full-sort Groups: 2  Sort Method: quicksort  Average Memory: 32kB  Peak Memory: 32kB
  Pre-sorted Groups: 1  Sort Method: quicksort  Average Memory: 47kB  Peak Memory: 47kB
  Buffers: shared hit=668 read=98
  ->  Nested Loop  (cost=3.03..162.05 rows=82 width=95) (actual time=2.008..6.292 rows=248 loops=1)
        Buffers: shared hit=659 read=98
        ->  Nested Loop  (cost=2.74..135.73 rows=82 width=72) (actual time=1.966..2.684 rows=248 loops=1)
              Buffers: shared hit=8 read=5
              ->  Index Scan using t1_q2_flight_airline_date on tab_air_flights f  (cost=0.28..2.50 rows=1 width=45) (actual time=1.248..1.257 rows=2 loops=1)
                    Index Cond: ((airline_id = 47) AND (departure >= '2023-01-01 00:00:00'::timestamp without time zone) AND (departure < '2023-04-01 00:00:00'::timestamp without time zone))
                    Buffers: shared hit=2 read=2
              ->  Bitmap Heap Scan on tab_air_bookings b  (cost=2.46..132.01 rows=122 width=35) (actual time=0.666..0.682 rows=124 loops=2)
                    Recheck Cond: (flight_id = f.flight_id)
                    Heap Blocks: exact=3
                    Buffers: shared hit=6 read=3
                    ->  Bitmap Index Scan on t1_q2_booking_flight  (cost=0.00..2.43 rows=122 width=0) (actual time=0.650..0.650 rows=124 loops=2)
                          Index Cond: (flight_id = f.flight_id)
                          Buffers: shared hit=3 read=3
        ->  Index Scan using t1_q2_passenger_id on tab_air_passengers p  (cost=0.29..0.31 rows=1 width=31) (actual time=0.014..0.014 rows=1 loops=248)
              Index Cond: (passenger_id = b.passenger_id)
              Buffers: shared hit=651 read=93
Planning:
  Buffers: shared hit=318 read=11
Planning Time: 5.697 ms
Execution Time: 7.036 ms
```

</details>

#### Conclusão

A leitura integral das duas tabelas maiores dá lugar a buscas das reservas dos dois voos e dos respectivos passageiros. A cardinalidade final continua em 248 linhas; a estimativa permanece em 82, mas a escolha de acesso já é mais barata. O Incremental Sort complementa a ordem parcial e mantém o requisito de ordenação.

#### Encerramento do experimento

```sql
DROP INDEX IF EXISTS t1_air.t1_q2_flight_airline_date;
DROP INDEX IF EXISTS t1_air.t1_q2_booking_flight;
DROP INDEX IF EXISTS t1_air.t1_q2_passenger_id;
```

### Q3 — Reservas de cinco passageiros

**Resultado:** 37 linhas. Custo 5039.71 → 58.16; execução 33.926 → 8.089 ms.

#### Consulta original

```sql
SELECT p.passenger_id, p.passportno, p.firstname, p.lastname,
       b.booking_id, b.seat, b.price,
       f.flight_id, f.flightno, f.departure, f.arrival,
       ao.iata AS origin_iata, ao.name AS origin_name,
       ad.iata AS destination_iata, ad.name AS destination_name
FROM t1_air.tab_air_passengers p
JOIN t1_air.tab_air_bookings b ON b.passenger_id = p.passenger_id
JOIN t1_air.tab_air_flights f ON f.flight_id = b.flight_id
JOIN t1_air.tab_air_airports ao ON ao.airport_id = f.from_airport_id
JOIN t1_air.tab_air_airports ad ON ad.airport_id = f.to_airport_id
WHERE p.passenger_id IN (18473, 26605, 13653, 12041, 17638)
ORDER BY p.passenger_id, f.departure;
```

#### Análise do plano original

O Seq Scan de passageiros encontra os cinco identificadores solicitados, mas precisa eliminar 36.090 registros. As junções seguintes leem todas as reservas, os voos e os dois aliases de aeroportos. O resultado contém 37 linhas, enquanto o plano estima 17.

O plano utiliza predominantemente Hash Join e termina com quicksort de 30 kB. Como o filtro inicial atinge apenas cinco passageiros, índices de identificação e de associação de reservas permitem que as junções cresçam a partir desse conjunto reduzido.

#### Estratégia adotada

Criar índices em passageiro, voo e aeroporto, mais índice composto de reservas por passageiro e voo. O predicado IN pode ser aplicado como condição do índice de passageiros; para cada um dos cinco identificadores, o índice de reservas recupera somente as associações correspondentes.

A consulta permanece igual. Todos os índices são usados. O plano passa a Nested Loop com Index Scan e Incremental Sort, aproveitando a ordem parcial pelo identificador do passageiro.

```sql
CREATE INDEX t1_q3_booking_passenger ON t1_air.tab_air_bookings (passenger_id, flight_id);
CREATE INDEX t1_q3_passenger_id ON t1_air.tab_air_passengers (passenger_id);
CREATE INDEX t1_q3_flight_id ON t1_air.tab_air_flights (flight_id);
CREATE INDEX t1_q3_airport_id ON t1_air.tab_air_airports (airport_id);
```

#### Planos registrados

<details>
<summary>Plano original — sem índices de otimização</summary>

```text
Sort  (cost=5039.67..5039.71 rows=17 width=113) (actual time=33.756..33.767 rows=37 loops=1)
  Sort Key: p.passenger_id, f.departure
  Sort Method: quicksort  Memory: 30kB
  Buffers: shared hit=2472
  ->  Hash Join  (cost=4837.66..5039.32 rows=17 width=113) (actual time=31.951..33.691 rows=37 loops=1)
        Hash Cond: (ad.airport_id = f.to_airport_id)
        Buffers: shared hit=2466
        ->  Seq Scan on tab_air_airports ad  (cost=0.00..164.54 rows=9854 width=19) (actual time=0.016..0.734 rows=9854 loops=1)
              Buffers: shared hit=66
        ->  Hash  (cost=4837.45..4837.45 rows=17 width=102) (actual time=31.846..31.854 rows=37 loops=1)
              Buckets: 1024  Batches: 1  Memory Usage: 14kB
              Buffers: shared hit=2400
              ->  Hash Join  (cost=4635.78..4837.45 rows=17 width=102) (actual time=29.987..31.816 rows=37 loops=1)
                    Hash Cond: (ao.airport_id = f.from_airport_id)
                    Buffers: shared hit=2400
                    ->  Seq Scan on tab_air_airports ao  (cost=0.00..164.54 rows=9854 width=19) (actual time=0.009..0.792 rows=9854 loops=1)
                          Buffers: shared hit=66
                    ->  Hash  (cost=4635.57..4635.57 rows=17 width=91) (actual time=29.904..29.907 rows=37 loops=1)
                          Buckets: 1024  Batches: 1  Memory Usage: 14kB
                          Buffers: shared hit=2334
                          ->  Hash Join  (cost=916.31..4635.57 rows=17 width=91) (actual time=6.310..29.860 rows=37 loops=1)
                                Hash Cond: (b.flight_id = f.flight_id)
                                Buffers: shared hit=2334
                                ->  Hash Join  (cost=866.61..4585.63 rows=17 width=58) (actual time=5.659..29.180 rows=37 loops=1)
                                      Hash Cond: (b.passenger_id = p.passenger_id)
                                      Buffers: shared hit=2318
                                      ->  Seq Scan on tab_air_bookings b  (cost=0.00..3260.44 rows=122244 width=35) (actual time=1.228..12.478 rows=122244 loops=1)
                                            Buffers: shared hit=2038
                                      ->  Hash  (cost=866.54..866.54 rows=5 width=31) (actual time=3.907..3.908 rows=5 loops=1)
                                            Buckets: 1024  Batches: 1  Memory Usage: 9kB
                                            Buffers: shared hit=280
                                            ->  Seq Scan on tab_air_passengers p  (cost=0.00..866.54 rows=5 width=31) (actual time=1.564..3.899 rows=5 loops=1)
                                                  Filter: (passenger_id = ANY ('{18473,26605,13653,12041,17638}'::bigint[]))
                                                  Rows Removed by Filter: 36090
                                                  Buffers: shared hit=280
                                ->  Hash  (cost=30.98..30.98 rows=1498 width=41) (actual time=0.636..0.636 rows=1498 loops=1)
                                      Buckets: 2048  Batches: 1  Memory Usage: 134kB
                                      Buffers: shared hit=16
                                      ->  Seq Scan on tab_air_flights f  (cost=0.00..30.98 rows=1498 width=41) (actual time=0.015..0.255 rows=1498 loops=1)
                                            Buffers: shared hit=16
Planning:
  Buffers: shared hit=249
Planning Time: 1.010 ms
Execution Time: 33.926 ms
```

</details>

<details>
<summary>Plano final — consulta escolhida com índices</summary>

```text
Incremental Sort  (cost=4.85..58.16 rows=17 width=113) (actual time=7.862..7.934 rows=37 loops=1)
  Sort Key: p.passenger_id, f.departure
  Presorted Key: p.passenger_id
  Full-sort Groups: 2  Sort Method: quicksort  Average Memory: 30kB  Peak Memory: 30kB
  Buffers: shared hit=368 read=38
  ->  Nested Loop  (cost=1.56..57.40 rows=17 width=113) (actual time=1.276..7.830 rows=37 loops=1)
        Buffers: shared hit=362 read=38
        ->  Nested Loop  (cost=1.27..50.90 rows=17 width=102) (actual time=1.257..7.584 rows=37 loops=1)
              Buffers: shared hit=263 read=26
              ->  Nested Loop  (cost=0.99..44.40 rows=17 width=91) (actual time=1.239..7.335 rows=37 loops=1)
                    Buffers: shared hit=164 read=14
                    ->  Nested Loop  (cost=0.71..39.21 rows=17 width=58) (actual time=1.210..7.170 rows=37 loops=1)
                          Buffers: shared hit=56 read=11
                          ->  Index Scan using t1_q3_passenger_id on tab_air_passengers p  (cost=0.29..9.07 rows=5 width=31) (actual time=0.616..3.072 rows=5 loops=1)
                                Index Cond: (passenger_id = ANY ('{18473,26605,13653,12041,17638}'::bigint[]))
                                Buffers: shared hit=10 read=5
                          ->  Index Scan using t1_q3_booking_passenger on tab_air_bookings b  (cost=0.42..5.99 rows=4 width=35) (actual time=0.782..0.815 rows=7 loops=5)
                                Index Cond: (passenger_id = p.passenger_id)
                                Buffers: shared hit=46 read=6
                    ->  Index Scan using t1_q3_flight_id on tab_air_flights f  (cost=0.28..0.30 rows=1 width=41) (actual time=0.003..0.004 rows=1 loops=37)
                          Index Cond: (flight_id = b.flight_id)
                          Buffers: shared hit=108 read=3
              ->  Index Scan using t1_q3_airport_id on tab_air_airports ao  (cost=0.29..0.37 rows=1 width=19) (actual time=0.006..0.006 rows=1 loops=37)
                    Index Cond: (airport_id = f.from_airport_id)
                    Buffers: shared hit=99 read=12
        ->  Index Scan using t1_q3_airport_id on tab_air_airports ad  (cost=0.29..0.37 rows=1 width=19) (actual time=0.006..0.006 rows=1 loops=37)
              Index Cond: (airport_id = f.to_airport_id)
              Buffers: shared hit=99 read=12
Planning:
  Buffers: shared hit=332 read=18
Planning Time: 6.177 ms
Execution Time: 8.089 ms
```

</details>

#### Conclusão

O plano percorre cinco passageiros e suas 37 reservas, em vez de ler integralmente as tabelas maiores para realizar Hash Join. As 37 linhas e as colunas selecionadas são preservadas. A ordenação por passageiro e partida também permanece.

#### Encerramento do experimento

```sql
DROP INDEX IF EXISTS t1_air.t1_q3_booking_passenger;
DROP INDEX IF EXISTS t1_air.t1_q3_passenger_id;
DROP INDEX IF EXISTS t1_air.t1_q3_flight_id;
DROP INDEX IF EXISTS t1_air.t1_q3_airport_id;
```

### Q4 — Passageiros com pelo menos dez reservas e total de 1.500

**Resultado:** 99 linhas. Custo 14488.08 → 6842.50; execução 209.120 → 128.314 ms.

#### Consulta original

```sql
SELECT p.passenger_id, p.firstname, p.lastname,
       COUNT(*) AS booking_count, SUM(b.price) AS total_paid,
       AVG(b.price) AS average_paid
FROM t1_air.tab_air_passengers p
JOIN t1_air.tab_air_bookings b ON b.passenger_id = p.passenger_id
GROUP BY p.passenger_id, p.firstname, p.lastname
HAVING COUNT(*) >= 10 AND SUM(b.price) >= 1500
ORDER BY total_paid DESC;
```

#### Análise do plano original

A consulta junta 122.244 reservas aos passageiros antes de agrupar por identificador, nome e sobrenome. O HashAggregate processa 34.870 grupos: 99 passam pelo HAVING e 34.771 são descartados. A estimativa da saída é de 4.011 grupos, muito superior às 99 linhas reais.

A agregação usa cinco lotes, cerca de 4.401 kB de memória e 6.936 kB em disco. O plano registra 499 blocos temporários lidos e 1.233 escritos. A ordenação final dos 99 resultados é pequena. Reduzir a largura e antecipar o HAVING antes da junção é mais relevante que otimizar o Sort final.

#### Estratégia adotada

Primeiro foram testados índices por passageiro em reservas e passageiros, mantendo a consulta original. O custo caiu para 12.726,20, mas a execução medida foi 228,260 ms, maior que os 209,120 ms originais. Logo, menor custo estimado nessa tentativa não significou menor tempo observado. O plano intermediário completo está abaixo.

A estratégia escolhida agrega as reservas somente por passenger_id e aplica COUNT/SUM/HAVING antes de consultar o nome. O índice de reservas fornece a ordem para GroupAggregate, evitando o HashAggregate com derramamento em disco. Somente os 99 grupos aprovados são levados à junção.

O índice de passageiros não é usado no plano final: a estimativa de 3.359 grupos ainda favorece Hash Join com Seq Scan. Isso não torna a criação do índice uma melhoria efetiva para esta consulta. O GroupAggregate via índice de reservas toca 122.215 buffers compartilhados; o ganho vem de evitar a agregação larga e os temporários, não de reduzir todos os acessos.

```sql
CREATE INDEX t1_q4_passenger_id ON t1_air.tab_air_passengers (passenger_id);
CREATE INDEX t1_q4_booking_passenger ON t1_air.tab_air_bookings (passenger_id);
```

#### Consulta reescrita

```sql
SELECT p.passenger_id, p.firstname, p.lastname,
       s.booking_count, s.total_paid, s.average_paid
FROM (
    SELECT passenger_id, COUNT(*) AS booking_count,
           SUM(price) AS total_paid, AVG(price) AS average_paid
    FROM t1_air.tab_air_bookings
    GROUP BY passenger_id
    HAVING COUNT(*) >= 10 AND SUM(price) >= 1500
) s
JOIN t1_air.tab_air_passengers p ON p.passenger_id = s.passenger_id
ORDER BY s.total_paid DESC;
```

#### Verificação da equivalência

A consulta abaixo retornou zero diferenças. A verificação compara as linhas e suas repetições; a ordenação está especificada no `ORDER BY` das consultas.

```sql
WITH original AS (
SELECT p.passenger_id, p.firstname, p.lastname,
       COUNT(*) AS booking_count, SUM(b.price) AS total_paid,
       AVG(b.price) AS average_paid
FROM t1_air.tab_air_passengers p
JOIN t1_air.tab_air_bookings b ON b.passenger_id = p.passenger_id
GROUP BY p.passenger_id, p.firstname, p.lastname
HAVING COUNT(*) >= 10 AND SUM(b.price) >= 1500
ORDER BY total_paid DESC
), otimizada AS (
SELECT p.passenger_id, p.firstname, p.lastname,
       s.booking_count, s.total_paid, s.average_paid
FROM (
    SELECT passenger_id, COUNT(*) AS booking_count,
           SUM(price) AS total_paid, AVG(price) AS average_paid
    FROM t1_air.tab_air_bookings
    GROUP BY passenger_id
    HAVING COUNT(*) >= 10 AND SUM(price) >= 1500
) s
JOIN t1_air.tab_air_passengers p ON p.passenger_id = s.passenger_id
ORDER BY s.total_paid DESC
)
SELECT COUNT(*) AS diferencas
FROM (
  (SELECT * FROM original EXCEPT ALL SELECT * FROM otimizada)
  UNION ALL
  (SELECT * FROM otimizada EXCEPT ALL SELECT * FROM original)
) comparacao;
```

#### Planos registrados

<details>
<summary>Plano original — sem índices de otimização</summary>

```text
Sort  (cost=14478.06..14488.08 rows=4011 width=93) (actual time=207.606..207.616 rows=99 loops=1)
  Sort Key: (sum(b.price)) DESC
  Sort Method: quicksort  Memory: 31kB
  Buffers: shared hit=2321, temp read=499 written=1233
  ->  HashAggregate  (cost=11844.80..14238.00 rows=4011 width=93) (actual time=146.730..207.520 rows=99 loops=1)
        Group Key: p.passenger_id, p.firstname, p.lastname
        Filter: ((count(*) >= 10) AND (sum(b.price) >= '1500'::numeric))
        Planned Partitions: 4  Batches: 5  Memory Usage: 4401kB  Disk Usage: 6936kB
        Rows Removed by Filter: 34771
        Buffers: shared hit=2318, temp read=499 written=1233
        ->  Hash Join  (cost=1092.14..6033.43 rows=122244 width=27) (actual time=14.580..78.023 rows=122244 loops=1)
              Hash Cond: (b.passenger_id = p.passenger_id)
              Buffers: shared hit=2318
              ->  Seq Scan on tab_air_bookings b  (cost=0.00..3260.44 rows=122244 width=14) (actual time=1.020..12.155 rows=122244 loops=1)
                    Buffers: shared hit=2038
              ->  Hash  (cost=640.95..640.95 rows=36095 width=21) (actual time=13.352..13.352 rows=36095 loops=1)
                    Buckets: 65536  Batches: 1  Memory Usage: 2418kB
                    Buffers: shared hit=280
                    ->  Seq Scan on tab_air_passengers p  (cost=0.00..640.95 rows=36095 width=21) (actual time=0.023..4.913 rows=36095 loops=1)
                          Buffers: shared hit=280
Planning:
  Buffers: shared hit=172
Planning Time: 0.818 ms
Execution Time: 209.120 ms
```

</details>

<details>
<summary>Plano da consulta original com os índices propostos</summary>

```text
Sort  (cost=12716.18..12726.20 rows=4011 width=93) (actual time=228.093..228.107 rows=99 loops=1)
  Sort Key: (sum(b.price)) DESC
  Sort Method: quicksort  Memory: 31kB
  Buffers: shared hit=122404 read=301
  ->  GroupAggregate  (cost=0.85..12476.12 rows=4011 width=93) (actual time=10.443..227.971 rows=99 loops=1)
        Group Key: p.passenger_id, p.firstname, p.lastname
        Filter: ((count(*) >= 10) AND (sum(b.price) >= '1500'::numeric))
        Rows Removed by Filter: 34771
        Buffers: shared hit=122401 read=301
        ->  Incremental Sort  (cost=0.85..10226.17 rows=122244 width=27) (actual time=0.241..168.653 rows=122244 loops=1)
              Sort Key: p.passenger_id, p.firstname, p.lastname
              Presorted Key: p.passenger_id
              Full-sort Groups: 3627  Sort Method: quicksort  Average Memory: 26kB  Peak Memory: 26kB
              Buffers: shared hit=122401 read=301
              ->  Merge Join  (cost=0.58..6861.53 rows=122244 width=27) (actual time=0.041..127.815 rows=122244 loops=1)
                    Merge Cond: (p.passenger_id = b.passenger_id)
                    Buffers: shared hit=122398 read=301
                    ->  Index Scan using t1_q4_passenger_id on tab_air_passengers p  (cost=0.29..938.86 rows=36095 width=21) (actual time=0.016..11.198 rows=36095 loops=1)
                          Buffers: shared hit=387 read=97
                    ->  Index Scan using t1_q4_booking_passenger on tab_air_bookings b  (cost=0.29..4304.55 rows=122244 width=14) (actual time=0.020..82.421 rows=122244 loops=1)
                          Buffers: shared hit=122011 read=204
Planning:
  Buffers: shared hit=212 read=8
Planning Time: 1.050 ms
Execution Time: 228.260 ms
```

</details>

<details>
<summary>Plano final — consulta escolhida com índices</summary>

```text
Sort  (cost=6834.10..6842.50 rows=3359 width=93) (actual time=127.532..127.542 rows=99 loops=1)
  Sort Key: s.total_paid DESC
  Sort Method: quicksort  Memory: 31kB
  Buffers: shared hit=122498
  ->  Hash Join  (cost=5901.66..6637.37 rows=3359 width=93) (actual time=120.668..127.444 rows=99 loops=1)
        Hash Cond: (p.passenger_id = s.passenger_id)
        Buffers: shared hit=122495
        ->  Seq Scan on tab_air_passengers p  (cost=0.00..640.95 rows=36095 width=21) (actual time=0.016..2.945 rows=36095 loops=1)
              Buffers: shared hit=280
        ->  Hash  (cost=5859.67..5859.67 rows=3359 width=80) (actual time=120.565..120.567 rows=99 loops=1)
              Buckets: 4096  Batches: 1  Memory Usage: 39kB
              Buffers: shared hit=122215
              ->  Subquery Scan on s  (cost=0.29..5859.67 rows=3359 width=80) (actual time=6.117..120.479 rows=99 loops=1)
                    Buffers: shared hit=122215
                    ->  GroupAggregate  (cost=0.29..5826.08 rows=3359 width=80) (actual time=6.115..120.451 rows=99 loops=1)
                          Group Key: tab_air_bookings.passenger_id
                          Filter: ((count(*) >= 10) AND (sum(tab_air_bookings.price) >= '1500'::numeric))
                          Rows Removed by Filter: 34771
                          Buffers: shared hit=122215
                          ->  Index Scan using t1_q4_booking_passenger on tab_air_bookings  (cost=0.29..4304.55 rows=122244 width=14) (actual time=0.025..73.148 rows=122244 loops=1)
                                Buffers: shared hit=122215
Planning:
  Buffers: shared hit=166
Planning Time: 1.438 ms
Execution Time: 128.314 ms
```

</details>

#### Conclusão

A reescrita preserva as 99 linhas, incluindo contagem, total e média, como confirmado por EXCEPT ALL nos dois sentidos. A junção ocorre depois da agregação. Não há blocos temporários no plano final; em contrapartida, a leitura ordenada pelo índice gera muitos acessos a buffers. A melhoria foi observada em custo e na execução registrada, com essa ressalva sobre o padrão de acesso.

#### Encerramento do experimento

```sql
DROP INDEX IF EXISTS t1_air.t1_q4_passenger_id;
DROP INDEX IF EXISTS t1_air.t1_q4_booking_passenger;
```

### Q5 — Voos de março de 2023 e valores das reservas

**Resultado:** 27 linhas. Custo 4687.34 → 280.57; execução 35.200 → 2.972 ms.

#### Consulta original

```sql
SELECT f.flight_id, f.flightno, f.departure, f.arrival,
       a.airline_name, ap.airplane_id, ap.capacity,
       apt.name AS airplane_type,
       ao.iata AS origin_iata, ao.name AS origin_name,
       ad.iata AS destination_iata, ad.name AS destination_name,
       COUNT(b.booking_id) AS booking_count,
       COALESCE(SUM(b.price), 0) AS total_paid,
       AVG(b.price) AS average_paid
FROM t1_air.tab_air_flights f
JOIN t1_air.tab_air_airlines a ON a.airline_id = f.airline_id
JOIN t1_air.tab_air_airplanes ap ON ap.airplane_id = f.airplane_id
JOIN t1_air.tab_air_airplane_types apt ON apt.airplane_type_id = ap.airplane_type_id
JOIN t1_air.tab_air_airports ao ON ao.airport_id = f.from_airport_id
JOIN t1_air.tab_air_airports ad ON ad.airport_id = f.to_airport_id
LEFT JOIN t1_air.tab_air_bookings b ON b.flight_id = f.flight_id
WHERE f.departure >= TIMESTAMP '2023-03-01'
  AND f.departure < TIMESTAMP '2023-04-01'
GROUP BY f.flight_id, f.flightno, f.departure, f.arrival,
         a.airline_name, ap.airplane_id, ap.capacity, apt.name,
         ao.iata, ao.name, ad.iata, ad.name
ORDER BY f.departure, f.flightno;
```

#### Análise do plano original

O filtro de março seleciona 27 voos, frente a 25 estimados. A Hash Right Join lê as 122.244 reservas e expande os voos em 1.507 linhas intermediárias; a ordenação de múltiplas colunas usa 265 kB antes do GroupAggregate. A raiz estima 2.040 grupos, mas produz 27.

As dimensões também são lidas sequencialmente. Das 27 partidas, 14 não possuem reservas; as demais somam 1.493 reservas. O LEFT JOIN é indispensável para manter os voos sem reservas. Neles, a contagem e o total devem ser zero, e a média permanece NULL, porque não há preços para calcular uma média.

#### Estratégia adotada

Criar índice em data de partida, índice de reservas por flight_id com INCLUDE(price), e índices nos identificadores das dimensões. A tentativa com índices, mas sem reescrever a consulta, reduz o custo para 2.410,27; ainda exige agrupamento sobre as linhas expandidas. O plano intermediário está abaixo.

A versão escolhida usa agregação correlacionada com LEFT JOIN LATERAL. Para cada um dos 27 voos selecionados, calcula as estatísticas de reservas antes de apresentá-las com as dimensões. O índice com price incluído permite Index Only Scan, com Heap Fetches igual a zero nesta execução.

COUNT(*) dentro da agregação correlacionada retorna zero quando não existem reservas; COALESCE converte o total nulo em zero. AVG é mantido nulo. A consulta conserva os 14 voos sem reservas. O índice de companhias não é usado: o otimizador prefere ler os 113 registros e fazer Hash Join. O acesso ao tipo de aeronave usa Memoize para reutilizar buscas.

```sql
CREATE INDEX t1_q5_flight_date ON t1_air.tab_air_flights (departure);
CREATE INDEX t1_q5_booking_flight_price ON t1_air.tab_air_bookings (flight_id) INCLUDE (price);
CREATE INDEX t1_q5_airline_id ON t1_air.tab_air_airlines (airline_id);
CREATE INDEX t1_q5_airplane_id ON t1_air.tab_air_airplanes (airplane_id);
CREATE INDEX t1_q5_type_id ON t1_air.tab_air_airplane_types (airplane_type_id);
CREATE INDEX t1_q5_airport_id ON t1_air.tab_air_airports (airport_id);
```

#### Consulta reescrita

```sql
SELECT f.flight_id, f.flightno, f.departure, f.arrival,
       a.airline_name, ap.airplane_id, ap.capacity,
       apt.name AS airplane_type,
       ao.iata AS origin_iata, ao.name AS origin_name,
       ad.iata AS destination_iata, ad.name AS destination_name,
       s.booking_count, COALESCE(s.total_paid, 0) AS total_paid,
       s.average_paid
FROM t1_air.tab_air_flights f
JOIN t1_air.tab_air_airlines a ON a.airline_id = f.airline_id
JOIN t1_air.tab_air_airplanes ap ON ap.airplane_id = f.airplane_id
JOIN t1_air.tab_air_airplane_types apt ON apt.airplane_type_id = ap.airplane_type_id
JOIN t1_air.tab_air_airports ao ON ao.airport_id = f.from_airport_id
JOIN t1_air.tab_air_airports ad ON ad.airport_id = f.to_airport_id
LEFT JOIN LATERAL (
    SELECT COUNT(*) AS booking_count, SUM(b.price) AS total_paid,
           AVG(b.price) AS average_paid
    FROM t1_air.tab_air_bookings b
    WHERE b.flight_id = f.flight_id
) s ON TRUE
WHERE f.departure >= TIMESTAMP '2023-03-01'
  AND f.departure < TIMESTAMP '2023-04-01'
ORDER BY f.departure, f.flightno;
```

#### Verificação da equivalência

A consulta abaixo retornou zero diferenças. A verificação compara as linhas e suas repetições; a ordenação está especificada no `ORDER BY` das consultas.

```sql
WITH original AS (
SELECT f.flight_id, f.flightno, f.departure, f.arrival,
       a.airline_name, ap.airplane_id, ap.capacity,
       apt.name AS airplane_type,
       ao.iata AS origin_iata, ao.name AS origin_name,
       ad.iata AS destination_iata, ad.name AS destination_name,
       COUNT(b.booking_id) AS booking_count,
       COALESCE(SUM(b.price), 0) AS total_paid,
       AVG(b.price) AS average_paid
FROM t1_air.tab_air_flights f
JOIN t1_air.tab_air_airlines a ON a.airline_id = f.airline_id
JOIN t1_air.tab_air_airplanes ap ON ap.airplane_id = f.airplane_id
JOIN t1_air.tab_air_airplane_types apt ON apt.airplane_type_id = ap.airplane_type_id
JOIN t1_air.tab_air_airports ao ON ao.airport_id = f.from_airport_id
JOIN t1_air.tab_air_airports ad ON ad.airport_id = f.to_airport_id
LEFT JOIN t1_air.tab_air_bookings b ON b.flight_id = f.flight_id
WHERE f.departure >= TIMESTAMP '2023-03-01'
  AND f.departure < TIMESTAMP '2023-04-01'
GROUP BY f.flight_id, f.flightno, f.departure, f.arrival,
         a.airline_name, ap.airplane_id, ap.capacity, apt.name,
         ao.iata, ao.name, ad.iata, ad.name
ORDER BY f.departure, f.flightno
), otimizada AS (
SELECT f.flight_id, f.flightno, f.departure, f.arrival,
       a.airline_name, ap.airplane_id, ap.capacity,
       apt.name AS airplane_type,
       ao.iata AS origin_iata, ao.name AS origin_name,
       ad.iata AS destination_iata, ad.name AS destination_name,
       s.booking_count, COALESCE(s.total_paid, 0) AS total_paid,
       s.average_paid
FROM t1_air.tab_air_flights f
JOIN t1_air.tab_air_airlines a ON a.airline_id = f.airline_id
JOIN t1_air.tab_air_airplanes ap ON ap.airplane_id = f.airplane_id
JOIN t1_air.tab_air_airplane_types apt ON apt.airplane_type_id = ap.airplane_type_id
JOIN t1_air.tab_air_airports ao ON ao.airport_id = f.from_airport_id
JOIN t1_air.tab_air_airports ad ON ad.airport_id = f.to_airport_id
LEFT JOIN LATERAL (
    SELECT COUNT(*) AS booking_count, SUM(b.price) AS total_paid,
           AVG(b.price) AS average_paid
    FROM t1_air.tab_air_bookings b
    WHERE b.flight_id = f.flight_id
) s ON TRUE
WHERE f.departure >= TIMESTAMP '2023-03-01'
  AND f.departure < TIMESTAMP '2023-04-01'
ORDER BY f.departure, f.flightno
)
SELECT COUNT(*) AS diferencas
FROM (
  (SELECT * FROM original EXCEPT ALL SELECT * FROM otimizada)
  UNION ALL
  (SELECT * FROM otimizada EXCEPT ALL SELECT * FROM original)
) comparacao;
```

#### Planos registrados

<details>
<summary>Plano original — sem índices de otimização</summary>

```text
GroupAggregate  (cost=4580.24..4687.34 rows=2040 width=174) (actual time=33.881..34.912 rows=27 loops=1)
  Group Key: f.departure, f.flightno, f.flight_id, f.arrival, a.airline_name, ap.airplane_id, ap.capacity, apt.name, ao.iata, ao.name, ad.iata, ad.name
  Buffers: shared hit=2236
  ->  Sort  (cost=4580.24..4585.34 rows=2040 width=116) (actual time=33.868..33.968 rows=1507 loops=1)
        Sort Key: f.departure, f.flightno, f.flight_id, f.arrival, a.airline_name, ap.airplane_id, ap.capacity, apt.name, ao.iata, ao.name, ad.iata, ad.name
        Sort Method: quicksort  Memory: 265kB
        Buffers: shared hit=2236
        ->  Hash Right Join  (cost=728.85..4468.10 rows=2040 width=116) (actual time=6.888..30.685 rows=1507 loops=1)
              Hash Cond: (b.flight_id = f.flight_id)
              Buffers: shared hit=2221
              ->  Seq Scan on tab_air_bookings b  (cost=0.00..3260.44 rows=122244 width=22) (actual time=1.049..10.906 rows=122244 loops=1)
                    Buffers: shared hit=2038
              ->  Hash  (cost=728.54..728.54 rows=25 width=102) (actual time=5.301..5.310 rows=27 loops=1)
                    Buckets: 1024  Batches: 1  Memory Usage: 12kB
                    Buffers: shared hit=183
                    ->  Hash Join  (cost=464.86..728.54 rows=25 width=102) (actual time=3.558..5.279 rows=27 loops=1)
                          Hash Cond: (f.airline_id = a.airline_id)
                          Buffers: shared hit=183
                          ->  Hash Join  (cost=461.32..724.65 rows=25 width=89) (actual time=3.456..5.166 rows=27 loops=1)
                                Hash Cond: (ad.airport_id = f.to_airport_id)
                                Buffers: shared hit=182
                                ->  Seq Scan on tab_air_airports ad  (cost=0.00..164.54 rows=9854 width=19) (actual time=0.023..0.744 rows=9854 loops=1)
                                      Buffers: shared hit=66
                                ->  Hash  (cost=461.01..461.01 rows=25 width=78) (actual time=3.315..3.321 rows=27 loops=1)
                                      Buckets: 1024  Batches: 1  Memory Usage: 12kB
                                      Buffers: shared hit=116
                                      ->  Hash Join  (cost=197.68..461.01 rows=25 width=78) (actual time=1.486..3.304 rows=27 loops=1)
                                            Hash Cond: (ao.airport_id = f.from_airport_id)
                                            Buffers: shared hit=116
                                            ->  Seq Scan on tab_air_airports ao  (cost=0.00..164.54 rows=9854 width=19) (actual time=0.008..0.770 rows=9854 loops=1)
                                                  Buffers: shared hit=66
                                            ->  Hash  (cost=197.36..197.36 rows=25 width=67) (actual time=1.401..1.405 rows=27 loops=1)
                                                  Buckets: 1024  Batches: 1  Memory Usage: 11kB
                                                  Buffers: shared hit=50
                                                  ->  Hash Join  (cost=188.98..197.36 rows=25 width=67) (actual time=1.315..1.387 rows=27 loops=1)
                                                        Hash Cond: (apt.airplane_type_id = ap.airplane_type_id)
                                                        Buffers: shared hit=50
                                                        ->  Seq Scan on tab_air_airplane_types apt  (cost=0.00..6.42 rows=342 width=18) (actual time=0.012..0.043 rows=342 loops=1)
                                                              Buffers: shared hit=3
                                                        ->  Hash  (cost=188.67..188.67 rows=25 width=57) (actual time=1.284..1.287 rows=27 loops=1)
                                                              Buckets: 1024  Batches: 1  Memory Usage: 11kB
                                                              Buffers: shared hit=47
                                                              ->  Hash Join  (cost=38.78..188.67 rows=25 width=57) (actual time=0.252..1.269 rows=27 loops=1)
                                                                    Hash Cond: (ap.airplane_id = f.airplane_id)
                                                                    Buffers: shared hit=47
                                                                    ->  Seq Scan on tab_air_airplanes ap  (cost=0.00..86.83 rows=5583 width=12) (actual time=0.011..0.473 rows=5583 loops=1)
                                                                          Buffers: shared hit=31
                                                                    ->  Hash  (cost=38.47..38.47 rows=25 width=49) (actual time=0.203..0.204 rows=27 loops=1)
                                                                          Buckets: 1024  Batches: 1  Memory Usage: 11kB
                                                                          Buffers: shared hit=16
                                                                          ->  Seq Scan on tab_air_flights f  (cost=0.00..38.47 rows=25 width=49) (actual time=0.026..0.192 rows=27 loops=1)
                                                                                Filter: ((departure >= '2023-03-01 00:00:00'::timestamp without time zone) AND (departure < '2023-04-01 00:00:00'::timestamp without time zone))
                                                                                Rows Removed by Filter: 1471
                                                                                Buffers: shared hit=16
                          ->  Hash  (cost=2.13..2.13 rows=113 width=21) (actual time=0.082..0.083 rows=113 loops=1)
                                Buckets: 1024  Batches: 1  Memory Usage: 15kB
                                Buffers: shared hit=1
                                ->  Seq Scan on tab_air_airlines a  (cost=0.00..2.13 rows=113 width=21) (actual time=0.021..0.055 rows=113 loops=1)
                                      Buffers: shared hit=1
Planning:
  Buffers: shared hit=255
Planning Time: 1.764 ms
Execution Time: 35.200 ms
```

</details>

<details>
<summary>Plano da consulta original com os índices propostos</summary>

```text
GroupAggregate  (cost=93.07..2410.27 rows=2040 width=174) (actual time=1.075..16.416 rows=27 loops=1)
  Group Key: f.departure, f.flightno, f.flight_id, f.arrival, a.airline_name, ap.airplane_id, ap.capacity, apt.name, ao.iata, ao.name, ad.iata, ad.name
  Buffers: shared hit=393 read=68
  ->  Incremental Sort  (cost=93.07..2308.27 rows=2040 width=116) (actual time=1.060..15.420 rows=1507 loops=1)
        Sort Key: f.departure, f.flightno, f.flight_id, f.arrival, a.airline_name, ap.airplane_id, ap.capacity, apt.name, ao.iata, ao.name, ad.iata, ad.name
        Presorted Key: f.departure
        Full-sort Groups: 14  Sort Method: quicksort  Average Memory: 29kB  Peak Memory: 29kB
        Pre-sorted Groups: 18  Sort Method: quicksort  Average Memory: 37kB  Peak Memory: 42kB
        Buffers: shared hit=393 read=68
        ->  Nested Loop Left Join  (cost=1.86..2217.49 rows=2040 width=116) (actual time=0.770..11.672 rows=1507 loops=1)
              Buffers: shared hit=378 read=68
              ->  Nested Loop  (cost=1.44..176.21 rows=25 width=102) (actual time=0.139..1.110 rows=27 loops=1)
                    Buffers: shared hit=298 read=37
                    ->  Nested Loop  (cost=1.16..124.40 rows=25 width=91) (actual time=0.115..0.880 rows=27 loops=1)
                          Buffers: shared hit=229 read=25
                          ->  Nested Loop  (cost=1.00..117.04 rows=25 width=78) (actual time=0.105..0.736 rows=27 loops=1)
                                Buffers: shared hit=187 read=25
                                ->  Nested Loop  (cost=0.72..65.22 rows=25 width=67) (actual time=0.086..0.519 rows=27 loops=1)
                                      Buffers: shared hit=117 read=14
                                      ->  Nested Loop  (cost=0.56..62.37 rows=25 width=57) (actual time=0.059..0.390 rows=27 loops=1)
                                            Buffers: shared hit=94 read=13
                                            ->  Index Scan using t1_q5_flight_date on tab_air_flights f  (cost=0.28..18.32 rows=25 width=49) (actual time=0.036..0.088 rows=27 loops=1)
                                                  Index Cond: ((departure >= '2023-03-01 00:00:00'::timestamp without time zone) AND (departure < '2023-04-01 00:00:00'::timestamp without time zone))
                                                  Buffers: shared hit=24 read=2
                                            ->  Index Scan using t1_q5_airplane_id on tab_air_airplanes ap  (cost=0.28..1.75 rows=1 width=12) (actual time=0.009..0.009 rows=1 loops=27)
                                                  Index Cond: (airplane_id = f.airplane_id)
                                                  Buffers: shared hit=70 read=11
                                      ->  Memoize  (cost=0.16..0.18 rows=1 width=18) (actual time=0.003..0.004 rows=1 loops=27)
                                            Cache Key: ap.airplane_type_id
                                            Cache Mode: logical
                                            Hits: 15  Misses: 12  Evictions: 0  Overflows: 0  Memory Usage: 2kB
                                            Buffers: shared hit=23 read=1
                                            ->  Index Scan using t1_q5_type_id on tab_air_airplane_types apt  (cost=0.15..0.17 rows=1 width=18) (actual time=0.005..0.005 rows=1 loops=12)
                                                  Index Cond: (airplane_type_id = ap.airplane_type_id)
                                                  Buffers: shared hit=23 read=1
                                ->  Index Scan using t1_q5_airport_id on tab_air_airports ao  (cost=0.29..2.06 rows=1 width=19) (actual time=0.007..0.007 rows=1 loops=27)
                                      Index Cond: (airport_id = f.from_airport_id)
                                      Buffers: shared hit=70 read=11
                          ->  Memoize  (cost=0.15..0.30 rows=1 width=21) (actual time=0.004..0.004 rows=1 loops=27)
                                Cache Key: f.airline_id
                                Cache Mode: logical
                                Hits: 6  Misses: 21  Evictions: 0  Overflows: 0  Memory Usage: 3kB
                                Buffers: shared hit=42
                                ->  Index Scan using t1_q5_airline_id on tab_air_airlines a  (cost=0.14..0.29 rows=1 width=21) (actual time=0.002..0.003 rows=1 loops=21)
                                      Index Cond: (airline_id = f.airline_id)
                                      Buffers: shared hit=42
                    ->  Index Scan using t1_q5_airport_id on tab_air_airports ad  (cost=0.29..2.06 rows=1 width=19) (actual time=0.007..0.008 rows=1 loops=27)
                          Index Cond: (airport_id = f.to_airport_id)
                          Buffers: shared hit=69 read=12
              ->  Index Scan using t1_q5_booking_flight_price on tab_air_bookings b  (cost=0.42..80.43 rows=122 width=22) (actual time=0.270..0.381 rows=55 loops=27)
                    Index Cond: (flight_id = f.flight_id)
                    Buffers: shared hit=80 read=31
Planning:
  Buffers: shared hit=369 read=18
Planning Time: 4.610 ms
Execution Time: 16.754 ms
```

</details>

<details>
<summary>Plano final — consulta escolhida com índices</summary>

```text
Sort  (cost=280.50..280.57 rows=25 width=174) (actual time=2.732..2.737 rows=27 loops=1)
  Sort Key: f.departure, f.flightno
  Sort Method: quicksort  Memory: 29kB
  Buffers: shared hit=389
  ->  Nested Loop Left Join  (cost=9.10..279.92 rows=25 width=174) (actual time=0.195..2.682 rows=27 loops=1)
        Buffers: shared hit=383
        ->  Nested Loop  (cost=4.83..172.74 rows=25 width=102) (actual time=0.175..2.003 rows=27 loops=1)
              Buffers: shared hit=294
              ->  Hash Join  (cost=4.67..169.88 rows=25 width=92) (actual time=0.160..1.929 rows=27 loops=1)
                    Hash Cond: (f.airline_id = a.airline_id)
                    Buffers: shared hit=270
                    ->  Nested Loop  (cost=1.13..165.99 rows=25 width=79) (actual time=0.093..1.845 rows=27 loops=1)
                          Buffers: shared hit=269
                          ->  Nested Loop  (cost=0.85..114.18 rows=25 width=68) (actual time=0.071..1.595 rows=27 loops=1)
                                Buffers: shared hit=188
                                ->  Nested Loop  (cost=0.56..62.37 rows=25 width=57) (actual time=0.058..1.423 rows=27 loops=1)
                                      Buffers: shared hit=107
                                      ->  Index Scan using t1_q5_flight_date on tab_air_flights f  (cost=0.28..18.32 rows=25 width=49) (actual time=0.044..0.076 rows=27 loops=1)
                                            Index Cond: ((departure >= '2023-03-01 00:00:00'::timestamp without time zone) AND (departure < '2023-04-01 00:00:00'::timestamp without time zone))
                                            Buffers: shared hit=26
                                      ->  Index Scan using t1_q5_airplane_id on tab_air_airplanes ap  (cost=0.28..1.75 rows=1 width=12) (actual time=0.049..0.049 rows=1 loops=27)
                                            Index Cond: (airplane_id = f.airplane_id)
                                            Buffers: shared hit=81
                                ->  Index Scan using t1_q5_airport_id on tab_air_airports ao  (cost=0.29..2.06 rows=1 width=19) (actual time=0.005..0.006 rows=1 loops=27)
                                      Index Cond: (airport_id = f.from_airport_id)
                                      Buffers: shared hit=81
                          ->  Index Scan using t1_q5_airport_id on tab_air_airports ad  (cost=0.29..2.06 rows=1 width=19) (actual time=0.007..0.007 rows=1 loops=27)
                                Index Cond: (airport_id = f.to_airport_id)
                                Buffers: shared hit=81
                    ->  Hash  (cost=2.13..2.13 rows=113 width=21) (actual time=0.053..0.054 rows=113 loops=1)
                          Buckets: 1024  Batches: 1  Memory Usage: 15kB
                          Buffers: shared hit=1
                          ->  Seq Scan on tab_air_airlines a  (cost=0.00..2.13 rows=113 width=21) (actual time=0.014..0.027 rows=113 loops=1)
                                Buffers: shared hit=1
              ->  Memoize  (cost=0.16..0.18 rows=1 width=18) (actual time=0.002..0.002 rows=1 loops=27)
                    Cache Key: ap.airplane_type_id
                    Cache Mode: logical
                    Hits: 15  Misses: 12  Evictions: 0  Overflows: 0  Memory Usage: 2kB
                    Buffers: shared hit=24
                    ->  Index Scan using t1_q5_type_id on tab_air_airplane_types apt  (cost=0.15..0.17 rows=1 width=18) (actual time=0.003..0.003 rows=1 loops=12)
                          Index Cond: (airplane_type_id = ap.airplane_type_id)
                          Buffers: shared hit=24
        ->  Aggregate  (cost=4.27..4.28 rows=1 width=72) (actual time=0.024..0.025 rows=1 loops=27)
              Buffers: shared hit=89
              ->  Index Only Scan using t1_q5_booking_flight_price on tab_air_bookings b  (cost=0.42..3.65 rows=122 width=6) (actual time=0.008..0.015 rows=55 loops=27)
                    Index Cond: (flight_id = f.flight_id)
                    Heap Fetches: 0
                    Buffers: shared hit=89
Planning:
  Buffers: shared hit=456
Planning Time: 3.444 ms
Execution Time: 2.972 ms
```

</details>

#### Conclusão

A versão reescrita produz exatamente as mesmas 27 linhas, incluindo os valores agregados e os voos sem reservas. O agrupamento amplo é substituído por 27 agregações seletivas, e a ordenação final usa 29 kB. O ganho é sustentado pelo Index Only Scan das reservas e pelas buscas das dimensões.

#### Encerramento do experimento

```sql
DROP INDEX IF EXISTS t1_air.t1_q5_flight_date;
DROP INDEX IF EXISTS t1_air.t1_q5_booking_flight_price;
DROP INDEX IF EXISTS t1_air.t1_q5_airline_id;
DROP INDEX IF EXISTS t1_air.t1_q5_airplane_id;
DROP INDEX IF EXISTS t1_air.t1_q5_type_id;
DROP INDEX IF EXISTS t1_air.t1_q5_airport_id;
```

### Q6 — Aeroportos nunca usados em voos

**Resultado:** 7382 linhas. Custo 1386.04 → 1270.98; execução 35.683 → 36.875 ms.

#### Consulta original

```sql
SELECT a.airport_id, a.iata, a.icao, a.name,
       g.city, g.country, g.latitude, g.longitude
FROM t1_air.tab_air_airports a
JOIN t1_air.tab_air_airports_geo g ON g.airport_id = a.airport_id
WHERE NOT EXISTS (
    SELECT 1 FROM t1_air.tab_air_flights f
    WHERE f.from_airport_id = a.airport_id
)
AND NOT EXISTS (
    SELECT 1 FROM t1_air.tab_air_flights f
    WHERE f.to_airport_id = a.airport_id
)
ORDER BY g.country, g.city, a.name;
```

#### Análise do plano original

O plano transforma os dois NOT EXISTS em duas Hash Anti Join. As duas leituras de voos examinam 1.498 registros cada; a primeira exclusão deixa 8.526 aeroportos, e a segunda deixa 7.382. A junção com localização e a ordenação por país, cidade e nome completam a consulta.

A estimativa de 7.384 linhas é muito próxima das 7.382 efetivas. A ordenação quicksort usa 764 kB e pesa bastante no custo total. Como quase três quartos dos aeroportos são retornados, uma leitura ampla de aeroportos e localização é coerente; criar índices nessas tabelas não garante benefício.

#### Estratégia adotada

Foram criados índices nas origens e destinos dos voos e no identificador de aeroporto em localização. Mantendo os dois NOT EXISTS, o custo cai apenas de 1.386,04 para 1.382,77 e a execução registrada sobe de 35,683 para 38,972 ms. A pequena diferença de custo não demonstra um ganho de tempo; o plano intermediário está abaixo.

A reescrita reúne origens e destinos com UNION e aplica uma única anti-junção. O plano usa dois Index Only Scan, Merge Append e Unique para formar os 2.472 aeroportos utilizados. O índice de localização não é usado: o conjunto de saída é amplo e Hash Join com Seq Scan continua mais adequado.

A raiz da versão reescrita estima 6.858 linhas, mais distante das 7.382 reais que a estimativa original. A ordenação de 764 kB permanece. Portanto, a redução do custo deve ser interpretada junto com a cardinalidade e o tempo observado.

```sql
CREATE INDEX t1_q6_flight_origin ON t1_air.tab_air_flights (from_airport_id);
CREATE INDEX t1_q6_flight_destination ON t1_air.tab_air_flights (to_airport_id);
CREATE INDEX t1_q6_geo_airport ON t1_air.tab_air_airports_geo (airport_id);
```

#### Consulta reescrita

```sql
SELECT a.airport_id, a.iata, a.icao, a.name,
       g.city, g.country, g.latitude, g.longitude
FROM t1_air.tab_air_airports a
JOIN t1_air.tab_air_airports_geo g ON g.airport_id = a.airport_id
WHERE NOT EXISTS (
    SELECT 1
    FROM (
        SELECT from_airport_id AS airport_id FROM t1_air.tab_air_flights
        UNION
        SELECT to_airport_id AS airport_id FROM t1_air.tab_air_flights
    ) used
    WHERE used.airport_id = a.airport_id
)
ORDER BY g.country, g.city, a.name;
```

#### Verificação da equivalência

A consulta abaixo retornou zero diferenças. A verificação compara as linhas e suas repetições; a ordenação está especificada no `ORDER BY` das consultas.

```sql
WITH original AS (
SELECT a.airport_id, a.iata, a.icao, a.name,
       g.city, g.country, g.latitude, g.longitude
FROM t1_air.tab_air_airports a
JOIN t1_air.tab_air_airports_geo g ON g.airport_id = a.airport_id
WHERE NOT EXISTS (
    SELECT 1 FROM t1_air.tab_air_flights f
    WHERE f.from_airport_id = a.airport_id
)
AND NOT EXISTS (
    SELECT 1 FROM t1_air.tab_air_flights f
    WHERE f.to_airport_id = a.airport_id
)
ORDER BY g.country, g.city, a.name
), otimizada AS (
SELECT a.airport_id, a.iata, a.icao, a.name,
       g.city, g.country, g.latitude, g.longitude
FROM t1_air.tab_air_airports a
JOIN t1_air.tab_air_airports_geo g ON g.airport_id = a.airport_id
WHERE NOT EXISTS (
    SELECT 1
    FROM (
        SELECT from_airport_id AS airport_id FROM t1_air.tab_air_flights
        UNION
        SELECT to_airport_id AS airport_id FROM t1_air.tab_air_flights
    ) used
    WHERE used.airport_id = a.airport_id
)
ORDER BY g.country, g.city, a.name
)
SELECT COUNT(*) AS diferencas
FROM (
  (SELECT * FROM original EXCEPT ALL SELECT * FROM otimizada)
  UNION ALL
  (SELECT * FROM otimizada EXCEPT ALL SELECT * FROM original)
) comparacao;
```

#### Planos registrados

<details>
<summary>Plano original — sem índices de otimização</summary>

```text
Sort  (cost=1367.58..1386.04 rows=7384 width=58) (actual time=34.366..34.985 rows=7382 loops=1)
  Sort Key: g.country, g.city, a.name
  Sort Method: quicksort  Memory: 764kB
  Buffers: shared hit=203
  ->  Hash Join  (cost=581.81..893.15 rows=7384 width=58) (actual time=7.433..11.941 rows=7382 loops=1)
        Hash Cond: (g.airport_id = a.airport_id)
        Buffers: shared hit=200
        ->  Seq Scan on tab_air_airports_geo g  (cost=0.00..200.54 rows=9854 width=38) (actual time=0.018..0.959 rows=9854 loops=1)
              Buffers: shared hit=102
        ->  Hash  (cost=489.51..489.51 rows=7384 width=24) (actual time=7.378..7.391 rows=7382 loops=1)
              Buckets: 8192  Batches: 1  Memory Usage: 461kB
              Buffers: shared hit=98
              ->  Hash Anti Join  (cost=99.41..489.51 rows=7384 width=24) (actual time=0.935..5.571 rows=7382 loops=1)
                    Hash Cond: (a.airport_id = f.from_airport_id)
                    Buffers: shared hit=98
                    ->  Hash Anti Join  (cost=49.70..335.17 rows=8526 width=24) (actual time=0.485..3.565 rows=8526 loops=1)
                          Hash Cond: (a.airport_id = f_1.to_airport_id)
                          Buffers: shared hit=82
                          ->  Seq Scan on tab_air_airports a  (cost=0.00..164.54 rows=9854 width=24) (actual time=0.011..0.819 rows=9854 loops=1)
                                Buffers: shared hit=66
                          ->  Hash  (cost=30.98..30.98 rows=1498 width=4) (actual time=0.459..0.463 rows=1498 loops=1)
                                Buckets: 2048  Batches: 1  Memory Usage: 69kB
                                Buffers: shared hit=16
                                ->  Seq Scan on tab_air_flights f_1  (cost=0.00..30.98 rows=1498 width=4) (actual time=0.016..0.212 rows=1498 loops=1)
                                      Buffers: shared hit=16
                    ->  Hash  (cost=30.98..30.98 rows=1498 width=4) (actual time=0.435..0.436 rows=1498 loops=1)
                          Buckets: 2048  Batches: 1  Memory Usage: 69kB
                          Buffers: shared hit=16
                          ->  Seq Scan on tab_air_flights f  (cost=0.00..30.98 rows=1498 width=4) (actual time=0.007..0.175 rows=1498 loops=1)
                                Buffers: shared hit=16
Planning:
  Buffers: shared hit=208
Planning Time: 0.778 ms
Execution Time: 35.683 ms
```

</details>

<details>
<summary>Plano da consulta original com os índices propostos</summary>

```text
Sort  (cost=1364.31..1382.77 rows=7384 width=58) (actual time=37.613..38.222 rows=7382 loops=1)
  Sort Key: g.country, g.city, a.name
  Sort Method: quicksort  Memory: 764kB
  Buffers: shared hit=179 read=4
  ->  Hash Join  (cost=578.55..889.88 rows=7384 width=58) (actual time=8.191..12.581 rows=7382 loops=1)
        Hash Cond: (g.airport_id = a.airport_id)
        Buffers: shared hit=176 read=4
        ->  Seq Scan on tab_air_airports_geo g  (cost=0.00..200.54 rows=9854 width=38) (actual time=0.015..0.928 rows=9854 loops=1)
              Buffers: shared hit=102
        ->  Hash  (cost=486.25..486.25 rows=7384 width=24) (actual time=8.113..8.119 rows=7382 loops=1)
              Buckets: 8192  Batches: 1  Memory Usage: 461kB
              Buffers: shared hit=74 read=4
              ->  Hash Anti Join  (cost=96.14..486.25 rows=7384 width=24) (actual time=1.029..6.314 rows=7382 loops=1)
                    Hash Cond: (a.airport_id = f.from_airport_id)
                    Buffers: shared hit=74 read=4
                    ->  Hash Anti Join  (cost=48.07..333.53 rows=8526 width=24) (actual time=0.563..4.255 rows=8526 loops=1)
                          Hash Cond: (a.airport_id = f_1.to_airport_id)
                          Buffers: shared hit=70 read=2
                          ->  Seq Scan on tab_air_airports a  (cost=0.00..164.54 rows=9854 width=24) (actual time=0.049..1.101 rows=9854 loops=1)
                                Buffers: shared hit=66
                          ->  Hash  (cost=29.35..29.35 rows=1498 width=4) (actual time=0.474..0.475 rows=1498 loops=1)
                                Buckets: 2048  Batches: 1  Memory Usage: 69kB
                                Buffers: shared hit=4 read=2
                                ->  Index Only Scan using t1_q6_flight_destination on tab_air_flights f_1  (cost=0.28..29.35 rows=1498 width=4) (actual time=0.019..0.210 rows=1498 loops=1)
                                      Heap Fetches: 0
                                      Buffers: shared hit=4 read=2
                    ->  Hash  (cost=29.35..29.35 rows=1498 width=4) (actual time=0.453..0.454 rows=1498 loops=1)
                          Buckets: 2048  Batches: 1  Memory Usage: 69kB
                          Buffers: shared hit=4 read=2
                          ->  Index Only Scan using t1_q6_flight_origin on tab_air_flights f  (cost=0.28..29.35 rows=1498 width=4) (actual time=0.019..0.208 rows=1498 loops=1)
                                Heap Fetches: 0
                                Buffers: shared hit=4 read=2
Planning:
  Buffers: shared hit=259 read=12
Planning Time: 1.297 ms
Execution Time: 38.972 ms
```

</details>

<details>
<summary>Plano final — consulta escolhida com índices</summary>

```text
Sort  (cost=1253.84..1270.98 rows=6858 width=58) (actual time=35.587..36.181 rows=7382 loops=1)
  Sort Key: g.country, g.city, a.name
  Sort Method: quicksort  Memory: 764kB
  Buffers: shared hit=183
  ->  Hash Join  (cost=510.79..816.86 rows=6858 width=58) (actual time=7.599..12.448 rows=7382 loops=1)
        Hash Cond: (g.airport_id = a.airport_id)
        Buffers: shared hit=180
        ->  Seq Scan on tab_air_airports_geo g  (cost=0.00..200.54 rows=9854 width=38) (actual time=0.015..1.014 rows=9854 loops=1)
              Buffers: shared hit=102
        ->  Hash  (cost=425.07..425.07 rows=6858 width=24) (actual time=7.547..7.552 rows=7382 loops=1)
              Buckets: 8192  Batches: 1  Memory Usage: 461kB
              Buffers: shared hit=78
              ->  Hash Right Anti Join  (cost=288.28..425.07 rows=6858 width=24) (actual time=4.545..5.831 rows=7382 loops=1)
                    Hash Cond: (tab_air_flights.from_airport_id = a.airport_id)
                    Buffers: shared hit=78
                    ->  Unique  (cost=0.57..96.16 rows=2996 width=4) (actual time=0.088..1.314 rows=2472 loops=1)
                          Buffers: shared hit=12
                          ->  Merge Append  (cost=0.57..88.67 rows=2996 width=4) (actual time=0.086..0.845 rows=2996 loops=1)
                                Sort Key: tab_air_flights.from_airport_id
                                Buffers: shared hit=12
                                ->  Index Only Scan using t1_q6_flight_origin on tab_air_flights  (cost=0.28..29.35 rows=1498 width=4) (actual time=0.058..0.242 rows=1498 loops=1)
                                      Heap Fetches: 0
                                      Buffers: shared hit=6
                                ->  Index Only Scan using t1_q6_flight_destination on tab_air_flights tab_air_flights_1  (cost=0.28..29.35 rows=1498 width=4) (actual time=0.027..0.215 rows=1498 loops=1)
                                      Heap Fetches: 0
                                      Buffers: shared hit=6
                    ->  Hash  (cost=164.54..164.54 rows=9854 width=24) (actual time=2.578..2.579 rows=9854 loops=1)
                          Buckets: 16384  Batches: 1  Memory Usage: 657kB
                          Buffers: shared hit=66
                          ->  Seq Scan on tab_air_airports a  (cost=0.00..164.54 rows=9854 width=24) (actual time=0.071..0.839 rows=9854 loops=1)
                                Buffers: shared hit=66
Planning:
  Buffers: shared hit=183
Planning Time: 0.909 ms
Execution Time: 36.875 ms
```

</details>

#### Conclusão

EXCEPT ALL confirma as mesmas 7.382 linhas. O custo cai aproximadamente 8,3%, mas a execução registrada passa de 35,683 para 36,875 ms. Houve melhoria do custo estimado e redução de alguns acessos a buffers, sem melhoria de tempo nesta medição. A estratégia é mantida como experimento válido, com essa limitação explícita.

#### Encerramento do experimento

```sql
DROP INDEX IF EXISTS t1_air.t1_q6_flight_origin;
DROP INDEX IF EXISTS t1_air.t1_q6_flight_destination;
DROP INDEX IF EXISTS t1_air.t1_q6_geo_airport;
```

### Q7 — Passageiros com sobrenome SMITH

**Resultado:** 268 linhas. Custo 1936.97 → 649.89; execução 32.065 → 4.596 ms.

#### Consulta original

```sql
SELECT p.passenger_id, p.passportno, p.firstname, p.lastname,
       d.street, d.city, d.zip, d.country
FROM t1_air.tab_air_passengers p
JOIN t1_air.tab_air_passengers_details d ON d.passenger_id = p.passenger_id
WHERE UPPER(p.lastname) = 'SMITH'
ORDER BY p.firstname, p.lastname;
```

#### Análise do plano original

O plano calcula UPPER(lastname) durante a leitura dos 36.095 passageiros, eliminando 35.827 registros e deixando 268. Depois, lê todos os 36.095 detalhes e faz Hash Join. A estimativa da saída é de 180 linhas; a ordenação final usa 49 kB.

Um índice simples em lastname não corresponde diretamente ao predicado UPPER(lastname). O alvo é uma estrutura que indexe exatamente essa expressão, além de um caminho seletivo para buscar a localização dos passageiros encontrados. O enunciado não informa a quantidade esperada para Q7.

#### Estratégia adotada

Criar índice de expressão em UPPER(lastname) e índice de detalhes por passenger_id. Após criar o índice de expressão, executar ANALYZE nos passageiros para coletar estatísticas da expressão. O SQL original é mantido, incluindo a pesquisa sem distinção de maiúsculas/minúsculas e a ordenação por nome e sobrenome.

O plano usa Bitmap Index Scan/Bitmap Heap Scan para encontrar os 268 passageiros e Index Scan para seus detalhes. A nova estimativa de 273 linhas fica mais próxima do resultado real. Os dois índices são utilizados.

```sql
CREATE INDEX t1_q7_passenger_upper_name ON t1_air.tab_air_passengers (UPPER(lastname));
CREATE INDEX t1_q7_details_passenger ON t1_air.tab_air_passengers_details (passenger_id);
ANALYZE t1_air.tab_air_passengers;
```

#### Planos registrados

<details>
<summary>Plano original — sem índices de otimização</summary>

```text
Sort  (cost=1936.52..1936.97 rows=180 width=71) (actual time=31.931..31.954 rows=268 loops=1)
  Sort Key: p.firstname, p.lastname
  Sort Method: quicksort  Memory: 49kB
  Buffers: shared hit=891
  ->  Hash Join  (cost=823.67..1929.78 rows=180 width=71) (actual time=22.853..31.450 rows=268 loops=1)
        Hash Cond: (d.passenger_id = p.passenger_id)
        Buffers: shared hit=888
        ->  Seq Scan on tab_air_passengers_details d  (cost=0.00..968.95 rows=36095 width=48) (actual time=0.017..3.748 rows=36095 loops=1)
              Buffers: shared hit=608
        ->  Hash  (cost=821.42..821.42 rows=180 width=31) (actual time=22.750..22.751 rows=268 loops=1)
              Buckets: 1024  Batches: 1  Memory Usage: 25kB
              Buffers: shared hit=280
              ->  Seq Scan on tab_air_passengers p  (cost=0.00..821.42 rows=180 width=31) (actual time=0.106..22.661 rows=268 loops=1)
                    Filter: (upper((lastname)::text) = 'SMITH'::text)
                    Rows Removed by Filter: 35827
                    Buffers: shared hit=280
Planning:
  Buffers: shared hit=206
Planning Time: 0.849 ms
Execution Time: 32.065 ms
```

</details>

<details>
<summary>Plano final — consulta escolhida com índices</summary>

```text
Sort  (cost=649.21..649.89 rows=273 width=71) (actual time=4.447..4.468 rows=268 loops=1)
  Sort Key: p.firstname, p.lastname
  Sort Method: quicksort  Memory: 49kB
  Buffers: shared hit=881 read=91
  ->  Nested Loop  (cost=3.80..638.16 rows=273 width=71) (actual time=0.166..3.972 rows=268 loops=1)
        Buffers: shared hit=878 read=91
        ->  Bitmap Heap Scan on tab_air_passengers p  (cost=3.51..195.08 rows=273 width=31) (actual time=0.138..0.578 rows=268 loops=1)
              Recheck Cond: (upper((lastname)::text) = 'SMITH'::text)
              Heap Blocks: exact=163
              Buffers: shared hit=163 read=2
              ->  Bitmap Index Scan on t1_q7_passenger_upper_name  (cost=0.00..3.44 rows=273 width=0) (actual time=0.113..0.113 rows=268 loops=1)
                    Index Cond: (upper((lastname)::text) = 'SMITH'::text)
                    Buffers: shared read=2
        ->  Index Scan using t1_q7_details_passenger on tab_air_passengers_details d  (cost=0.29..1.61 rows=1 width=48) (actual time=0.012..0.012 rows=1 loops=268)
              Index Cond: (passenger_id = p.passenger_id)
              Buffers: shared hit=715 read=89
Planning:
  Buffers: shared hit=251 read=5 dirtied=7
Planning Time: 5.895 ms
Execution Time: 4.596 ms
```

</details>

#### Conclusão

A pesquisa de sobrenome deixa de aplicar UPPER durante uma varredura completa e passa a usar o índice da expressão. As 268 linhas permanecem, com suas informações de identificação e localização. A redução do custo e da execução registrada é consistente com os acessos seletivos.

#### Encerramento do experimento

```sql
DROP INDEX IF EXISTS t1_air.t1_q7_passenger_upper_name;
DROP INDEX IF EXISTS t1_air.t1_q7_details_passenger;
```

## 5. Conclusão geral

Conseguimos executar a atividade no Supabase, utilizando os recursos nativos do PostgreSQL para observar e comparar os planos. Os ganhos mais claros vieram de filtros seletivos com índices, antecipação da agregação e um índice correspondente à expressão `UPPER(lastname)`.

Apenas criar um índice não garante seu uso nem uma redução do tempo: alguns índices de dimensões não foram escolhidos pelo planejador, a tentativa intermediária de Q4 piorou o tempo observado e Q6 apresentou apenas redução de custo estimado. Esses casos foram registrados para manter a análise coerente com os planos.

O banco ficou preparado e populado para repetir as consultas. Os índices experimentais foram removidos ao final. Esta publicação contém exclusivamente a documentação em Markdown. Os dez TSVs foram mantidos localmente para uma possível publicação posterior, após a confirmação da origem dos registros. O relatório PDF produzido na atividade foi mantido fora da publicação.
