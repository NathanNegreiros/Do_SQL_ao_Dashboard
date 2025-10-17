# Dashboard de Vendas e Usuários

## Visão Geral

Este projeto tem como objetivo construir dois dashboards analíticos — **Vendas** e **Usuários** — utilizando consultas SQL otimizadas no **Google BigQuery**. O foco foi simular um dia completo de análise de dados, incluindo **planejamento de métricas, levantamento de KPIs, modelagem de dados, criação de queries SQL e construção do dashboard no Looker Studio**.

O objetivo era criar dashboards que fornecessem insights estratégicos para os stakeholders, contemplando indicadores financeiros, operacionais e de comportamento de usuários.

---

## Simulação do Processo de Trabalho

O projeto foi desenvolvido simulando o fluxo de trabalho real de um analista de dados:

1. **Reunião com Stakeholders**  
   - Simulou-se uma reunião com os stakeholders, em que foram levantadas as necessidades de negócio.
  
2. **Planejamento e definição de KPIs**  
   - Listagem completa das métricas necessárias, como **vendas totais, receita, lucro, ticket médio, margem de lucro, taxa de cancelamento, crescimento mensal**, entre outros.
   - Para usuários, métricas como **total de usuários, pedidos por usuário, usuários recorrentes vs novos, valor gasto por usuário, performance por canal de tráfego, distribuição por faixa etária**.
   - Definição da **fonte de cada KPI** dentro do banco de dados disponível, analisando quais tabelas e colunas poderiam ser utilizadas.

3. **Análise e exploração das tabelas disponíveis**  
   - Verificação das tabelas `users`, `order_items` e `products`.
   - Identificação de quais campos poderiam ser utilizados diretamente e quais precisariam ser calculados.

4. **Estruturação das queries SQL**  
   - Criação de **CTEs** para organizar os dados antes da agregação final.
   - Realização de **joins**, agregações e cálculos intermediários para cada métrica.
   - Criação de tabelas intermediárias que alimentariam o dashboard.

5. **Criação das tabelas finais para o dashboard**  
   - Tabela de vendas agregada (`tabela_vendas`)  
   - Tabela de vendas mensal para análise de crescimento (`tabela_vendas_mensal`)  
   - Tabela consolidada de usuários (`Clientes_dashhboard`)  
   - Todas as tabelas foram criadas para facilitar a visualização e reduzir complexidade no dashboard.

6. **Desenvolvimento do dashboard**  
   - Definição do layout e organização dos gráficos.
   - Conexão das tabelas do BigQuery no **Looker Studio**.
   - Configuração de métricas calculadas no dashboard (como margem de lucro, taxa de cancelamento).

7. **Validação e conferência**  
   - Verificação se os números do dashboard batiam com os cálculos SQL.
   - Ajustes nas métricas, garantindo consistência e confiabilidade.

---

## Estrutura do Projeto

O projeto foi dividido em duas partes principais:

### 1. Dashboard de Vendas

Inclui indicadores financeiros e operacionais, como **receita, lucro, ticket médio, margem de lucro, taxa de cancelamento e crescimento mensal**.

**Imagem do Dashboard de Vendas**  
![Dashboard de Vendas](Dashboard%20Vendas.jpg)  

Link do dashboard: [Visualizar Dashboard](https://lookerstudio.google.com/reporting/bbecfe04-8aad-4952-8ec8-ce2fd434480c)

---

### 2. Dashboard de Usuários

Inclui indicadores de comportamento e perfil de clientes, como **recorrência, faixa etária, origem de tráfego e ranking de clientes**.

**Imagem do Dashboard de Usuários**  
![Dashboard de Usuários](Dashboard%20Clientes.jpg)  

Link do dashboard: [Visualizar Dashboard](https://lookerstudio.google.com/u/2/reporting/bbecfe04-8aad-4952-8ec8-ce2fd434480c/page/p_l5ixr7k9wd)

---

## Planejamento e Definição de Indicadores

### Dashboard de Vendas

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

### Dashboard de Usuários

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

### Tabela de Vendas Mensal

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

* **Dashboard de Vendas:** insights financeiros e operacionais.
* **Dashboard de Usuários:** análise do comportamento, perfil demográfico e performance por canal de aquisição.

---

## Principais Aprendizados

* Planejamento detalhado das métricas antes de criar tabelas e dashboards.
* Estruturação de pipelines SQL modulares e reutilizáveis.
* Uso de funções de janela (`lag`, `row_number`) para métricas dinâmicas.
* Criação de dashboards claros e orientados à tomada de decisão.

---

## Ferramentas Utilizadas

* **Google BigQuery** — modelagem e processamento de dados.
* **Looker Studio** — visualização e dashboards.
* **SQL (Standard SQL)** — consultas e agregações.
* **Google Cloud Platform (GCP)** — ambiente de execução.




