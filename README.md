# Dashboard de vendas e metas em Power BI

Relatório de acompanhamento de uma rede fictícia de quatro lojas, com 26 medidas DAX que vão do básico à inteligência de tempo. Material de aula e referência para portfólio.

**👉 Aluno: comece pelo [passo a passo do exercício](PASSO_A_PASSO.md)**, que ensina a construir este dashboard do zero, com conferência de resultados em cada etapa.

> 📸 Prints das páginas em breve.

## Páginas

| Página | Responde |
|---|---|
| **Início** | Menu de navegação |
| **Metas de vendas** | Cada loja bateu a meta? Como faturamento e meta evoluem mês a mês e no acumulado do ano? Em que meses houve desvio? |
| **Visão geral de vendas** | Quanto cada loja vende? Em que horas saem mais pedidos? Quanto tempo leva a entrega? Quais categorias dão mais margem? |

As duas páginas de análise têm filtros de **ano** e **filial**.

## Modelo de dados

Modelo estrela, com duas tabelas fato e quatro dimensões.

```
dCalendario ─┐
dLoja ───────┼──► fVendas ◄── dProdutos
dFuncionario ┘        
dLoja ──────────► fMetas
```

| Tabela | Tipo | Conteúdo |
|---|---|---|
| `fVendas` | Fato | 3.000 vendas de 2019 e 2020 |
| `fMetas` | Fato | Meta mensal por loja |
| `dLoja` | Dimensão | 4 lojas |
| `dProdutos` | Dimensão | 20 produtos em 4 categorias |
| `dFuncionario` | Dimensão | 12 vendedores |
| `dCalendario` | Dimensão | Tabela calculada com `CALENDAR` |

## Medidas DAX

**Agregações básicas**

| Medida | Função principal | O que calcula |
|---|---|---|
| `faturamento` | `SUMX` | Soma de valor unitário × quantidade, linha a linha |
| `Lucro` | `SUMX` | Soma de (valor unitário − custo) × quantidade |
| `margem_lucro` | `DIVIDE` | Lucro ÷ faturamento |
| `ticket_medio` | `AVERAGEX` | Valor médio por pedido |
| `qtde_pedidos` | `COUNTROWS` | Número de vendas |
| `produtos_dist` | `DISTINCTCOUNT` | Produtos diferentes vendidos |
| `total_funcionarios` | `COUNT` | Número de vendedores |
| `media_salario` | `AVERAGE` | Salário médio |

**Metas**

| Medida | Função principal | O que calcula |
|---|---|---|
| `meta` | `SUM` | Meta de faturamento |
| `dif (real-meta)` | subtração de medidas | Faturamento − meta |
| `%Meta` | `DIVIDE` | Desvio percentual em relação à meta |
| `metaYTD` | `DATESYTD` | Meta acumulada no ano |

**Contexto de filtro**

| Medida | Função principal | O que calcula |
|---|---|---|
| `QtdeProd_Maior2` | `FILTER` | Pedidos com 3 ou mais unidades |
| `faturamento_lojaES` | `CALCULATE` | Faturamento só da Filial ES |
| `faturamento_2Lojas` | `CALCULATE` + `IN` | Faturamento de duas filiais |
| `faturamento_total` | `ALL` | Faturamento ignorando todos os filtros |
| `faturamento_ALLEXCEPT` | `ALLEXCEPT` | Faturamento mantendo só o filtro de ano |
| `%Fat` | variáveis + `ALL` | Participação da loja no total |
| `faturamento_prod` | `CALCULATE` + `FILTER` | Faturamento dos produtos com mais de 5 pedidos |
| `qtde_entregas` | `USERELATIONSHIP` | Pedidos pela data de entrega (relacionamento inativo) |

**Inteligência de tempo**

| Medida | Função principal | O que calcula |
|---|---|---|
| `faturamentoYTD` | `DATESYTD` | Faturamento acumulado no ano |
| `faturamentoLY` | `SAMEPERIODLASTYEAR` | Faturamento do mesmo período do ano anterior |
| `Ano Atual vs Ano Anterior` | subtração de medidas | Variação entre os dois anos |
| `LucroLY` | `SAMEPERIODLASTYEAR` | Lucro do mesmo período do ano anterior |

**Tempo de entrega**

| Medida | Função principal | O que calcula |
|---|---|---|
| `tempo_medio_entrega (hs)` | `AVERAGEX` + `DATEDIFF` | Horas médias entre venda e entrega |
| `tempo_entrega (hh:mm:ss)` | variáveis | O mesmo tempo, formatado como hh:mm:ss |

Há também duas colunas calculadas em `dFuncionario`: `Class Vendas` (`IF`) e `Satisfacao` (`SWITCH(TRUE())`).

## Como abrir

1. Baixe [`dashboard-vendas.pbix`](dashboard-vendas.pbix) e [`BD.xlsx`](BD.xlsx).
2. Abra o `.pbix` no Power BI Desktop. Os dados já vêm carregados.
3. Para atualizar: **Transformar dados → Configurações da fonte de dados → Alterar fonte**, aponte para o seu `BD.xlsx` e clique em **Atualizar**.

## Faça a sua versão

1. Crie uma medida de faturamento por vendedor e um ranking com `RANKX`.
2. Adicione uma página de RH usando `dFuncionario` (salário, satisfação, performance).
3. Troque a meta fixa por uma meta de crescimento sobre o ano anterior.
4. Refaça o layout com a sua identidade visual.

## Créditos

- **Base de dados:** fictícia, gerada para este projeto. Veja [`datasets-pratica/bd-vendas-lojas`](https://github.com/iasenai1111-ux/datasets-pratica/tree/main/bd-vendas-lojas).
- **Layout e documentação:** próprios.
- **Modelo e conjunto de medidas:** baseados nos estudos do curso *Fundamentos de DAX*, da Empowerdata.
