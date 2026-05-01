# 📊 Prova Analista BI & MIS — Parte 1/2: Modelagem Dimensional em PostgreSQL

> Solução completa de Data Warehouse para análise de casos de atendimento, em PostgreSQL. Este projeto é parte de um processo seletivo para analista de dados. Esta é a **Parte 1 de 2** do projeto — abrange a camada de dados (SQL). A **Parte 2** cobrirá o dashboard desenvolvido no Power BI.

---

## 📁 Estrutura do Repositório

```
.
├── BaseCasos.csv                   # Base de dados de origem (amostra)
├── dim_fato_procedure_v3.txt       # Script SQL principal
└── README.md
```

---

## 📋 Das Instruções do Projeto

- Subir a base `BaseCasos.csv` para o banco de dados PostgreSQL. A tabela no banco deve ter o nome de `hist_casos_trabalhados`.
- Criar tabelas dimensão com base na tabela `hist_casos_trabalhados` criada anteriormente:
  - `dim_calendario` — com base na coluna `Data_Hora_Criação`
  - `dim_funcionario` — com base na coluna `NomeAgen`
  - `dim_supervisor` — com base na coluna `NomeSupe`
  - `dim_motivo_chamado` — com base na coluna `Motivo_Chamador`
  - `dim_status` — com base na coluna `Status`
  - `dim_canal_entrada` — com base na coluna `Canal_Entrada`
  - `dim_pais` — com base na coluna `País`
- Criar **tabela fato** com base na `hist_casos_trabalhados`, contendo os campos identificadores das dimensões e os campos necessários para calcular os indicadores:
  - % Resolução = soma do total de "Yes" / (soma de "Yes" + "No")
  - Total de casos fechados (status "Done")
  - Total de casos abertos
  - Tempo médio para atualização em horas
  - Tempo médio para fechamento em horas
  - A tabela fato não deve trazer o `id_caso` — a sumarização é feita via `GROUP BY`.
- Criar uma **procedure** que atualize as tabelas dimensão e fato.
- Criar um **dashboard no Power BI** com os indicadores acima *(Parte 2)*.

---

## 💡 Considerações sobre a Escolha do Modelo e Data Profiling

O requisito de não incluir uma coluna FK com o ID do caso, mais a sumarização por `GROUP BY`, caracteriza o **achatamento dos dados** — processo que reduz a quantidade de linhas na tabela fato, aumentando o desempenho e indicando um foco analítico mais amplo.

Desta forma, optou-se por diminuir a granularidade da `dim_calendario` ao formato **DATA** (`DD-MM-AAAA`), uma vez que manter o formato original (`DD-MM-AAAA HH:MM`) geraria tantas linhas na tabela fato quanto uma tabela não sumarizada.

A exploração dos dados (data profiling) elucidou os seguintes pontos:

- **Colunas com valores nulos pontuais** (`pais` e `motivo_chamado`): optou-se pelo tratamento com *placeholders* `'(NAO INFORMADO)'`, e não o descarte das linhas, para manter a integridade dos totais, a transparência aos stakeholders e a preservação do evento.

- **Coluna `resolucao` com valores nulos altíssimos**: notadamente uma regra de negócio — casos ainda sem uma resolução final (`YES` ou `NO`) ficam nulos. Além das colunas de contagem de `YES` e `NO` exigidas, incluiu-se a coluna `qtd_total_casos` para guardar as parciais e permitir o cálculo da proporção pela ferramenta de BI.

- **Placeholder `'01/01/1900 00:00'` na coluna `data_hora_fechamento`**: outra regra de negócio para casos não fechados. Utilizou-se `NULLIF` para converter esse valor em `NULL` e evitar distorções nos cálculos de indicadores de tempo.

---

## 📂 Base de Dados de Origem

**Arquivo:** `BaseCasos.csv` — separado por `;`, encoding UTF-8 com BOM.

| Coluna | Tipo (origem) | Descrição |
|---|---|---|
| `Id_Caso` | texto | Identificador único do caso |
| `País` | texto | País de origem do atendimento |
| `Canal_Entrada` | texto | Canal pelo qual o caso chegou (ex: CHAT) |
| `Status` | texto | Situação do caso (ex: Done) |
| `Resolução` | texto | Se o caso foi resolvido (`Yes` / `No`) |
| `Motivo_Chamador` | texto | Motivo do contato |
| `NomeAgen` | texto | Nome do agente responsável |
| `NomeSupe` | texto | Nome do supervisor |
| `Data_Hora_Criação` | texto | Timestamp de abertura (`DD/MM/YYYY HH:MM`) |
| `Data_Hora_Atualização` | texto | Timestamp da última atualização |
| `Data_Hora_Fechamento` | texto | Timestamp de fechamento |

---

## 🏗️ Arquitetura — Star Schema

O modelo segue o padrão **Star Schema**, com uma tabela fato central conectada a 7 dimensões:

```
                    ┌─────────────────┐
                    │  dim_calendario │
                    └────────┬────────┘
                             │
┌──────────────┐    ┌────────▼────────┐    ┌───────────────────┐
│   dim_pais   ├────►                 ◄────┤ dim_canal_entrada  │
└──────────────┘    │   fato_casos    │    └───────────────────┘
                    │                 │
┌──────────────┐    │  (indicadores   │    ┌───────────────────┐
│  dim_status  ├────►   de negócio)   ◄────┤ dim_motivo_chamado│
└──────────────┘    │                 │    └───────────────────┘
                    └────────┬────────┘
                    ┌────────┴────────┐
          ┌─────────┴──────┐  ┌───────┴────────┐
          │ dim_funcionario│  │ dim_supervisor  │
          └────────────────┘  └────────────────┘
```

---

## 🔧 Explicação do Script SQL

### 1. Tabela de Staging — `hist_casos_trabalhados`

```sql
CREATE TABLE hist_casos_trabalhados (
    id_caso               VARCHAR(255),
    pais                  VARCHAR(255),
    canal_entrada         VARCHAR(255),
    status                VARCHAR(255),
    resolucao             VARCHAR(255),
    motivo_chamador       VARCHAR(255),
    nome_agente           VARCHAR(255),
    nome_supe             VARCHAR(255),
    data_hora_criacao     VARCHAR(255),
    data_hora_atualizacao VARCHAR(255),
    data_hora_fechamento  VARCHAR(255)
);
```

**Por que todos os campos são `VARCHAR`?**  
A ingestão direta de CSV pode trazer datas em formato textual (`DD/MM/YYYY HH:MM`) que o PostgreSQL não aceita automaticamente como `TIMESTAMP`. Usar `VARCHAR` na staging evita rejeição de linhas durante o `COPY` e centraliza o tratamento no SQL.

---

### 2. Tabelas Dimensão

Todas as dimensões — com exceção da `dim_calendario` — seguem o mesmo padrão de construção em 3 passos:

```
CREATE TABLE dim_X AS SELECT DISTINCT ... → ADD PK SERIAL → ADD UNIQUE CONSTRAINT
```

**Tratamento padronizado aplicado em todas as dimensões:**
```sql
UPPER(TRIM(COALESCE(coluna, '(NAO INFORMADO)')))
```
- `COALESCE` → substitui `NULL` pelo placeholder `'(NAO INFORMADO)'`
- `TRIM` → remove espaços extras antes e depois
- `UPPER` → padroniza tudo em maiúsculas, evitando duplicatas por capitalização diferente

#### `dim_pais`
Dimensão de países. Gerada a partir dos valores distintos da coluna `pais`.  
Chave: `id_pais SERIAL PRIMARY KEY` | Unicidade: `uk_pais UNIQUE (pais)`

#### `dim_canal_entrada`
Dimensão dos canais de entrada do caso (ex: CHAT, EMAIL, TELEFONE).  
Chave: `id_canal_entrada SERIAL PRIMARY KEY`

#### `dim_status`
Dimensão de status do caso (ex: `DONE`, `IN_PROGRESS`).  
Chave: `id_status SERIAL PRIMARY KEY`

#### `dim_motivo_chamado`
Dimensão dos motivos de contato informados pelo chamador.  
Chave: `id_motivo_chamado SERIAL PRIMARY KEY`

#### `dim_funcionario`
Dimensão dos agentes de atendimento (`nome_agente`).  
Chave: `id_funcionario SERIAL PRIMARY KEY`

#### `dim_supervisor`
Dimensão dos supervisores (`nome_supe`).  
Chave: `id_supervisor SERIAL PRIMARY KEY`

#### `dim_calendario`

Optou-se por incluir vários atributos de data para facilitar o *slicing* e o desempenho na fase de BI. Utiliza-se uma CTE (`sub`) para converter a coluna raw em `TIMESTAMP`. Converte-se então em `CHAR` para reduzir a granularidade (`YYYYMMDD`) e por fim em `INT`, formato utilizado como PK.

```sql
-- CTE interna que converte o VARCHAR da staging em TIMESTAMP
FROM (
    SELECT DISTINCT
        TO_TIMESTAMP(data_hora_criacao, 'DD/MM/YYYY HH24:MI') AS data_ref
    FROM hist_casos_trabalhados
    WHERE data_hora_criacao IS NOT NULL
) sub

-- A PK é adicionada após a criação da tabela
ALTER TABLE dim_calendario
    ADD CONSTRAINT pk_id_calendario PRIMARY KEY (id_calendario);
```

Atributos gerados automaticamente a partir da data:

| Coluna | Exemplo | Descrição |
|---|---|---|
| `id_calendario` | `20210316` | Chave no formato `YYYYMMDD` (INT) |
| `data` | `2021-03-16` | Data como tipo `DATE` |
| `ano` | `2021` | Ano numérico |
| `numero_trimestre` | `1` | Trimestre (1–4) |
| `trimestre` | `T1` | Label do trimestre |
| `numero_mes` | `3` | Mês numérico |
| `nome_mes` | `Março` | Nome por extenso (locale) |
| `abrev_mes` | `Mar` | Abreviação (3 letras) |
| `ano_mes` | `2021-03` | Período no formato `YYYY-MM` |
| `numero_semana_ano` | `11` | Semana ISO do ano |
| `numero_dia_semana` | `2` | Dia da semana (0=Dom, 6=Sáb) |
| `nome_dia_semana` | `Tuesday` | Nome do dia (locale) |
| `dia` | `16` | Dia do mês |
| `dia_do_ano` | `75` | Dia sequencial do ano |
| `flag_fim_de_semana` | `FALSE` | `TRUE` se Sábado ou Domingo |

---

### 3. Tabela Fato — `fato_casos`

A tabela fato é construída em **4 etapas** organizadas com CTEs:

#### Etapa 1 — CTE `base`: Conversão e Normalização

```sql
WITH base AS (
    SELECT
        TO_TIMESTAMP(data_hora_criacao,    'DD/MM/YYYY HH24:MI') AS data_hora_criacao_ts,
        TO_TIMESTAMP(data_hora_atualizacao,'DD/MM/YYYY HH24:MI') AS data_hora_atualizacao_ts,
        NULLIF(
            TO_TIMESTAMP(data_hora_fechamento, 'DD/MM/YYYY HH24:MI'),
            TO_TIMESTAMP('01/01/1900 00:00',   'DD/MM/YYYY HH24:MI')
        ) AS data_hora_fechamento_ts,
        UPPER(TRIM(COALESCE(pais,          '(NAO INFORMADO)'))) AS pais_norm,
        UPPER(TRIM(COALESCE(canal_entrada, '(NAO INFORMADO)'))) AS canal_entrada_norm,
        -- ... demais campos normalizados
    FROM hist_casos_trabalhados h
    WHERE h.data_hora_criacao IS NOT NULL
)
```

- As datas são convertidas de `VARCHAR` para `TIMESTAMP` com `TO_TIMESTAMP`.
- `NULLIF(..., '01/01/1900 00:00')` trata o placeholder de fechamento vazio, transformando-o em `NULL` — casos ainda abertos não possuem data de fechamento real.
- Todos os campos categóricos recebem `UPPER(TRIM(COALESCE(...)))` para alinhamento com as dimensões.

#### Etapa 2 — CTE `com_fks`: Resolução das Chaves Estrangeiras

```sql
com_fks AS (
    SELECT
        b.*,
        p.id_pais,
        ce.id_canal_entrada,
        s.id_status,
        mc.id_motivo_chamado,
        f.id_funcionario,
        sp.id_supervisor,
        TO_CHAR(data_hora_criacao_ts, 'YYYYMMDD')::INT AS id_calendario
    FROM base b
    LEFT JOIN dim_pais           p  ON p.pais           = b.pais_norm
    LEFT JOIN dim_canal_entrada  ce ON ce.canal_entrada  = b.canal_entrada_norm
    LEFT JOIN dim_funcionario    f  ON f.funcionario     = b.nome_agente_norm
    LEFT JOIN dim_motivo_chamado mc ON mc.motivo_chamado = b.motivo_chamador_norm
    LEFT JOIN dim_status         s  ON s.status          = b.status_norm
    LEFT JOIN dim_supervisor     sp ON sp.supervisor     = b.nome_supe_norm
)
```

- `LEFT JOIN` garante que nenhuma linha seja perdida, mesmo que haja um valor sem correspondência na dimensão.
- O `id_calendario` é gerado com a mesma lógica da `dim_calendario` (`TO_CHAR → INT`), garantindo a integridade do JOIN.

#### Etapa 3 — `GROUP BY` e cálculo dos indicadores

```sql
SELECT
    id_pais, id_canal_entrada, id_status,
    id_motivo_chamado, id_funcionario, id_supervisor, id_calendario,

    COUNT(*)                                      AS qtd_total_casos,
    COUNT(*) FILTER(WHERE resolucao_norm = 'YES') AS resolucao_yes,
    COUNT(*) FILTER(WHERE resolucao_norm = 'NO')  AS resolucao_no,
    COUNT(*) FILTER(WHERE status_norm = 'DONE')   AS total_casos_fechados,
    COUNT(*) FILTER(WHERE status_norm <> 'DONE')  AS total_casos_abertos,

    -- Soma e contagem separadas para permitir médias corretas
    -- em qualquer nível de drill-down no Power BI (média = soma / qtd via DAX)
    SUM(EXTRACT(EPOCH FROM (data_hora_atualizacao_ts - data_hora_criacao_ts)) / 3600)
        FILTER(WHERE data_hora_atualizacao_ts IS NOT NULL)::NUMERIC AS soma_horas_atualizacao,

    SUM(EXTRACT(EPOCH FROM (data_hora_fechamento_ts - data_hora_criacao_ts)) / 3600)
        FILTER(WHERE data_hora_fechamento_ts IS NOT NULL)::NUMERIC  AS soma_horas_fechamento,

    COUNT(EXTRACT(EPOCH FROM (data_hora_atualizacao_ts - data_hora_criacao_ts)) / 3600)
        FILTER(WHERE data_hora_atualizacao_ts IS NOT NULL)::NUMERIC AS qtd_com_atualizacao,

    COUNT(EXTRACT(EPOCH FROM (data_hora_fechamento_ts - data_hora_criacao_ts)) / 3600)
        FILTER(WHERE data_hora_fechamento_ts IS NOT NULL)::NUMERIC  AS qtd_com_fechamento

FROM com_fks
GROUP BY id_calendario, id_pais, id_canal_entrada,
         id_status, id_motivo_chamado, id_funcionario, id_supervisor
```

**Por que `soma_horas` e `qtd_com_*` em vez de `AVG` direto?**  
Ao agrupar no banco, médias pré-calculadas não podem ser re-agregadas corretamente no Power BI quando o usuário faz drill-down por outra dimensão. Armazenar a soma e a contagem separadamente permite que o DAX recalcule a média correta em qualquer nível: `média = soma / qtd`.

**Cálculo do tempo em horas:**
```sql
EXTRACT(EPOCH FROM (timestamp_final - timestamp_inicial)) / 3600
-- EPOCH extrai a diferença em segundos; dividir por 3600 converte para horas
```

#### Etapa 4 — PK, Índices e Foreign Keys

```sql
-- Chave primária surrogate
ALTER TABLE fato_casos ADD COLUMN id_fato SERIAL PRIMARY KEY;

-- Índices para performance nas junções
CREATE INDEX idx_fato_calendario ON fato_casos (id_calendario);
CREATE INDEX idx_fato_pais       ON fato_casos (id_pais);
-- ... demais índices

-- Constraints de FK explícitas
ALTER TABLE fato_casos
    ADD CONSTRAINT fk_fato_calendario
        FOREIGN KEY (id_calendario) REFERENCES dim_calendario (id_calendario),
    ADD CONSTRAINT fk_fato_pais
        FOREIGN KEY (id_pais) REFERENCES dim_pais (id_pais);
    -- ... demais FKs
```

---

### 4. Procedure de Atualização — `sp_atualiza_dw()`

```sql
CREATE PROCEDURE sp_atualiza_dw()
LANGUAGE plpgsql AS $$
BEGIN
    -- 9 etapas de atualização com RAISE NOTICE para logging
    ...
EXCEPTION
    WHEN OTHERS THEN
        RAISE EXCEPTION '✖ sp_atualiza_dw falhou: % — %', SQLERRM, SQLSTATE;
END;
$$;
```

A procedure executa um **full refresh** de todo o DW em 9 etapas sequenciais:

| Etapa | Ação |
|---|---|
| 1/9 | `TRUNCATE fato_casos` — limpa a fato primeiro (respeita FK constraints) |
| 2/9 | Full refresh de `dim_calendario` |
| 3/9 | Full refresh de `dim_pais` |
| 4/9 | Full refresh de `dim_canal_entrada` |
| 5/9 | Full refresh de `dim_status` |
| 6/9 | Full refresh de `dim_motivo_chamado` |
| 7/9 | Full refresh de `dim_funcionario` |
| 8/9 | Full refresh de `dim_supervisor` |
| 9/9 | Reinsere `fato_casos` com toda a lógica de CTEs |

**Detalhe importante — ordem do `TRUNCATE`:**  
A fato é truncada **antes** das dimensões porque as FKs da fato referenciam as PKs das dimensões. Truncar uma dimensão com linhas referenciadas na fato geraria erro de violação de constraint. Limpando a fato primeiro, as dimensões podem ser limpas e recarregadas livremente.

**Tratamento de erros:**  
O bloco `EXCEPTION WHEN OTHERS` captura qualquer falha, relançando uma mensagem clara com `SQLERRM` (mensagem do erro) e `SQLSTATE` (código padrão SQL), facilitando o diagnóstico.

---

## 📐 Indicadores de Negócio

| Indicador | Colunas na Fato | Fórmula (DAX / SQL) |
|---|---|---|
| % Resolução | `resolucao_yes`, `resolucao_no` | `resolucao_yes / (resolucao_yes + resolucao_no)` |
| Total de Casos Fechados | `total_casos_fechados` | Casos com `status = 'DONE'` |
| Total de Casos Abertos | `total_casos_abertos` | Casos com `status <> 'DONE'` |
| Tempo Médio de Atualização (h) | `soma_horas_atualizacao`, `qtd_com_atualizacao` | `soma / qtd` |
| Tempo Médio de Fechamento (h) | `soma_horas_fechamento`, `qtd_com_fechamento` | `soma / qtd` |

---

## ▶️ Como Executar

### Pré-requisitos
- PostgreSQL 13+
- Acesso a um banco de dados (ex: `casos_atendimento`)

### Passo a Passo

```bash
# 1. Conecte ao banco
psql -U seu_usuario -d casos_atendimento

# 2. Execute o script completo
\i dim_fato_procedure_v3.txt

# 3. Carregue o CSV na tabela staging
\COPY hist_casos_trabalhados FROM 'BaseCasos.csv'
    WITH (FORMAT CSV, HEADER TRUE, DELIMITER ';', ENCODING 'UTF8');

# 4. Execute a procedure para popular o DW
CALL sp_atualiza_dw();
```

> **Atenção:** o CSV utiliza BOM (`UTF-8 BOM`). Se o `\COPY` retornar erro na primeira coluna, utilize `ENCODING 'UTF8BOM'` ou remova o BOM com um editor de texto antes da importação.

---

## 🔗 Parte 2

A **Parte 2** deste projeto contém o dashboard desenvolvido no **Power BI**, conectado ao banco PostgreSQL construído aqui, com os seguintes painéis:

- Visão geral de casos (abertos × fechados)
- % de Resolução por canal, país e período
- Tempo médio de atualização e fechamento
- Ranking de agentes e supervisores

---

## 🛠️ Tecnologias

- **PostgreSQL 13+** — banco de dados relacional
- **PL/pgSQL** — linguagem procedural para a stored procedure
- **Power BI** — visualização (Parte 2)
