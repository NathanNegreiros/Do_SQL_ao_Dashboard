# Dashboard de Vendas e Usuários

## Visão Geral

Este projeto tem como objetivo construir dois dashboards analíticos — **Vendas** e **Usuários** — utilizando consultas SQL otimizadas no **Google BigQuery**. O foco foi estruturar e modelar os dados de forma profissional, simulando o processo de criação de indicadores de negócio solicitados por stakeholders, desde o planejamento das métricas até a visualização final no **Looker Studio**.

---

## Estrutura do Projeto

O projeto é dividido em duas partes principais:

### 1. Dashboard de Vendas

Baseado em duas tabelas criadas a partir de dados transacionais brutos. Inclui indicadores financeiros e operacionais, como **receita, lucro e taxa de cancelamento**.

**Imagem do Dashboard de Vendas**  
![Dashboard de Vendas](Dashboard%20Vendas.jpg)  

Link do dashboard: [Visualizar Dashboard](https://lookerstudio.google.com/reporting/bbecfe04-8aad-4952-8ec8-ce2fd434480c)

---

### 2. Dashboard de Usuários

Baseado em uma única tabela consolidada construída a partir da integração de múltiplas tabelas. Apresenta indicadores de comportamento e perfil de cliente, como **recorrência, faixa etária e origem de tráfego**.

**Imagem do Dashboard de Usuários**  
![Dashboard de Usuários](Dashboard%20Clientes.jpg)

Link do dashboard: [Visualizar Dashboard](https://lookerstudio.google.com/u/2/reporting/bbecfe04-8aad-4952-8ec8-ce2fd434480c/page/p_l5ixr7k9wd)


---

## Planejamento e Definição de Indicadores

Antes da implementação, foi realizada a estruturação dos indicadores, definindo quais métricas seriam incluídas e de onde viriam dentro das tabelas disponíveis. Essa etapa simulou uma reunião com stakeholders, em que foram levantadas as principais necessidades de negócio.

### Indicadores do Dashboard de Vendas

| Indicador | Descrição | Fonte / Cálculo |
|-----------|-----------|----------------|
| Vendas Totais | Quantidade de vendas no período | order_id (tabela order_items) |
| Receita Total | Soma dos valores vendidos | sale_price (tabela order_items) |
| Lucro Total | Receita - Custo | sale_price - cost |
| Percentual de Custo | Custo / Receita | Calculado no dashboard |
| Margem de Lucro | Lucro / Receita | Calculado no dashboard |
| Pedidos Cancelados | Total de pedidos com status “Cancelled” | status |
| Taxa de Cancelamento | Cancelados / Total de pedidos | Calculado no dashboard |
| Crescimento Mensal de Vendas | Variação percentual mês a mês | tabela_vendas_mensal |
| Ticket Médio | Receita / Quantidade de pedidos | Calculado no dashboard |
| Vendas por Status | Comparativo de status de pedidos | status |

---

### Indicadores do Dashboard de Usuários

| Indicador | Descrição | Fonte / Cálculo |
|-----------|-----------|----------------|
| Total de Pedidos | Quantidade total de pedidos realizados | order_id |
| Total de Usuários | Quantidade de usuários únicos | user_id |
| Pedidos por Usuário | Pedidos / Usuários | Calculado no dashboard |
| Valor Gasto por Usuário | Receita / Usuário | sum(sale_price) |
| Usuários Novos vs Recorrentes | Classificação por número de compras | row_number() over(partition by user_id) |
| Performance por Tipo de Tráfego | Análise por traffic_source | Tabela users |
| Top Clientes | Ranking por valor total gasto | Agrupamento por user_id |
| Distribuição por Faixa Etária | Agrupamento condicional por idade | CASE na query |

---

## Modelagem de Dados

As queries abaixo foram utilizadas para criar as tabelas base que alimentam os dashboards. Todas foram executadas no **BigQuery**, dentro do projeto `projectnathanportifolio`.

### Tabela de Vendas

```sql
create or replace table vendas-451017.sandbox.tabela_vendas as (
with tab as (
  select
    t1.order_id,
    t1.status,
    cast(t1.created_at as date) as data_pedido,
    cast(t1.shipped_at as date) as data_envio,
    cast(t1.delivered_at as date) as data_entrega,
    t1.sale_price,
    t2.cost,
    (t1.sale_price - t2.cost) as lucro
  from vendas-451017.sandbox.order_items as t1
  left join vendas-451017.sandbox.products as t2
  on t1.product_id = t2.id
)
select
  order_id,
  status,
  data_pedido,
  data_envio,
  data_entrega,
  sum(sale_price) as valor_venda,
  sum(cost) as custo,
  sum(lucro) as lucro
from tab
group by 1, 2, 3, 4, 5
order by 1
);
````

---

### Tabela de Vendas Mensal (Crescimento Mês a Mês)

```sql
create or replace table vendas-451017.sandbox.tabela_vendas_mensal as (
with tab as (
  select  
    date_trunc(data_pedido, month) as mes,
    sum(valor_venda) as receita
  from vendas-451017.sandbox.tabela_vendas 
  group by 1
)
select
  *,
  coalesce((receita - lag(receita) over(order by mes asc)) / lag(receita) over(order by mes asc), 0) as mom
from tab
order by mes asc
);
```

---

### Tabela de Usuários Consolidada

```sql
create or replace table projectnathanportifolio.Vendas.Clientes_dashhboard as (
with geral as (
  select 
    t1.id as user_id,
    t2.order_id,
    date_trunc(t2.created_at, month) as mes_pedido,
    t1.age,
    t1.traffic_source,
    t2.sale_price
  from projectnathanportifolio.Vendas.users as t1
  inner join projectnathanportifolio.Vendas.order_items as t2
  on t1.id = t2.user_id
),

agregacao as (
  select 
    geral.user_id,
    order_id,
    mes_pedido,
    age,
    traffic_source,
    sum(sale_price) as preco
  from geral
  group by 1, 2, 3, 4, 5
),

classificacoes as (
  select *,
  row_number() over(partition by user_id order by mes_pedido asc) as n_compras,
  case
    when age between 12 and 18 then '12-18'
    when age between 19 and 25 then '19-25'
    when age between 26 and 32 then '26-32'
    when age between 33 and 39 then '33-39'
    when age between 40 and 46 then '40-46'
    when age between 47 and 56 then '47-56'
    else '57+'
  end as faixa_idade
  from agregacao
)

select *,
case
  when n_compras = 1 then 'novo'
  else 'recorrente'
end as classificacao_cliente
from classificacoes
);
```

---

## Resultados

Os dashboards foram desenvolvidos no **Looker Studio**, conectando diretamente as tabelas criadas no **BigQuery**. O resultado foi a entrega de duas visões estratégicas:

### Dashboard de Vendas

Indicadores de desempenho financeiro e operacional — **receita, lucro, margem, cancelamentos, crescimento mensal**, entre outros.

**Imagem complementar do Dashboard de Vendas (visão detalhada)**
![Dashboard de Vendas Detalhado](caminho-da-imagem-aqui)

---

### Dashboard de Usuários

Insights sobre o comportamento dos clientes, perfil demográfico e performance por canal de aquisição.

**Imagem complementar do Dashboard de Usuários (visão analítica)**
![Dashboard de Usuários Detalhado](caminho-da-imagem-aqui)

---

## Principais Aprendizados

* Planejamento e modelagem de indicadores antes da implementação é essencial.
* Construção de pipelines SQL modulares, permitindo a reutilização de tabelas e métricas.
* Aplicação de funções de janela para cálculo de métricas dinâmicas (`lag`, `row_number`).
* Criação de dashboards orientados a negócio, com foco em clareza e tomada de decisão.

---

## Ferramentas Utilizadas

* **Google BigQuery** — modelagem e processamento de dados.
* **Looker Studio** — criação dos dashboards e visualizações.
* **SQL (Standard SQL)** — consultas, cálculos e agregações.
* **Google Cloud Platform (GCP)** — ambiente de execução e armazenamento.

