# Passo a passo: construa o dashboard do zero

**Objetivo:** ao final, você terá construído um dashboard de vendas e metas com modelo estrela, tabela calendário e medidas DAX, pronto para o seu portfólio.

| | |
|---|---|
| **Nível** | Básico a intermediário |
| **Pré-requisito** | Power BI Desktop instalado e noção de Excel |
| **Tempo estimado** | 4 a 6 horas, que podem ser divididas pelas partes abaixo |
| **Material** | [`BD.xlsx`](BD.xlsx) |

**Contexto:** você é analista de uma rede de quatro lojas. A diretoria quer saber, toda segunda-feira, se cada loja está batendo a meta e como as vendas estão evoluindo. Hoje isso é feito à mão em planilha. Sua tarefa é entregar um relatório que responda a essas perguntas sozinho.

> Use o [`dashboard-vendas.pbix`](dashboard-vendas.pbix) como gabarito: tente fazer primeiro e abra o arquivo só para conferir.

---

## Parte 1: importar e tratar os dados (Power Query)

1. Abra o Power BI Desktop → **Obter dados → Pasta de trabalho do Excel** → selecione o `BD.xlsx`.
2. Marque as cinco abas (`fVendas`, `dFuncionario`, `dLoja`, `dProdutos`, `Metas`) e clique em **Transformar Dados**. Não clique em Carregar ainda.

### 1.1 `dLoja` e `dProdutos`

Confira se a primeira linha virou cabeçalho. Se as colunas aparecerem como `Column1`, `Column2`, use **Usar a Primeira Linha como Cabeçalho**. Tipos: `codigo_produto` como número inteiro e o restante como texto.

### 1.2 `dFuncionario`

1. Remova as colunas `dt_demissao`, `motivo_saida` e `PerfScoreID`, que não serão usadas.
2. Confira os tipos: datas como **Data**, `salario_mensal` como número inteiro, `pesquisa_engajamento` e `indice_satisfacao` como número decimal.

### 1.3 `fVendas`

A aba traz data e hora juntas e algumas colunas repetidas das dimensões.

1. Remova `Nome Funcionario`, `Cargo` e `Categoria`. Essas informações já existem em `dFuncionario` e `dProdutos`; na tabela fato fica só o código.
2. Selecione `dt_venda` → **Adicionar Coluna → Hora → Somente Hora**. Renomeie a nova coluna para `hr_venda`.
3. Com `hr_venda` selecionada → **Adicionar Coluna → Hora → Hora**. Renomeie para `int_hr_venda` (a hora cheia, de 8 a 19).
4. Selecione `dt_venda` → **Transformar → Data → Somente Data**.
5. Repita os passos 2 e 4 para `dt_entrega`, criando `hr_entrega`.

### 1.4 `Metas` (renomeie a consulta para `fMetas`)

A aba está em formato de relatório: um título na primeira linha, uma coluna por mês e uma linha de total. Para análise, precisamos de uma linha por loja e por mês.

1. Se o Power BI tiver criado sozinho as etapas "Cabeçalhos Promovidos" e "Tipo Alterado", apague-as no painel **Etapas Aplicadas**.
2. **Remover Linhas → Remover Linhas Principais → 1** (tira o título "Meta de Faturamento").
3. **Usar a Primeira Linha como Cabeçalho**.
4. Na coluna `Orcamento`, desmarque `Total` no filtro.
5. Remova as colunas `Meta 2019` e `Meta 2020`.
6. Selecione `Orcamento` → **Transformar → Transformar Colunas em Linhas → Transformar Outras Colunas em Linhas**.
7. Renomeie: `Orcamento` → `cod_loja`, `Atributo` → `data`, `Valor` → `valor_meta`.
8. Tipos: `data` como **Data** e `valor_meta` como **Número Decimal Fixo**.

> 🔴 **Erro comum:** deixar a linha `Total` na tabela. A soma das metas sai dobrada, porque o total entra como se fosse mais uma loja.

Clique em **Fechar e Aplicar**.

> 🟢 **Checkpoint 1:** `fVendas` com 3.000 linhas, `fMetas` com 96 (4 lojas × 24 meses), `dFuncionario` com 12, `dProdutos` com 20 e `dLoja` com 4.

---

## Parte 2: tabela calendário

Na faixa **Modelagem → Nova tabela**:

```dax
dCalendario =
CALENDAR(
    DATE(YEAR(MIN(fVendas[dt_venda])), 1, 1),
    DATE(YEAR(MAX(fVendas[dt_entrega])), 12, 31)
)
```

Depois, com a tabela selecionada, crie as colunas (**Nova coluna**), uma por vez:

```dax
ano = YEAR(dCalendario[Date])
mes_num = MONTH(dCalendario[Date])
nome_mes = FORMAT(dCalendario[Date], "mmm")
trimestre = "T" & FORMAT(dCalendario[Date], "Q")
dia_semana = FORMAT(dCalendario[Date], "ddd")
mes_ano = EOMONTH(dCalendario[Date], 0)
```

Por fim:

1. Selecione `nome_mes` → **Classificar por coluna → mes_num**.
2. **Ferramentas da tabela → Marcar como tabela de data**, usando a coluna `Date`.

> 🔴 **Erro comum:** esquecer de classificar `nome_mes`. Os meses aparecem em ordem alfabética (abr, ago, dez...) nos gráficos.

> 🟢 **Checkpoint 2:** `dCalendario` com 731 linhas, de 01/01/2019 a 31/12/2020.

---

## Parte 3: relacionamentos

Na **Exibição de Modelo**, arraste a coluna de uma tabela para a outra:

| De (muitos) | Para (um) | Situação |
|---|---|---|
| `fVendas[codigo_loja]` | `dLoja[codigo_loja]` | Ativo |
| `fVendas[codigo_produto]` | `dProdutos[codigo_produto]` | Ativo |
| `fVendas[matricula_funcionario]` | `dFuncionario[matricula_funcionario]` | Ativo |
| `fVendas[dt_venda]` | `dCalendario[Date]` | Ativo |
| `fVendas[dt_entrega]` | `dCalendario[Date]` | **Inativo** (linha tracejada) |
| `fMetas[cod_loja]` | `dLoja[codigo_loja]` | Ativo |
| `fMetas[data]` | `dCalendario[Date]` | Ativo |

Só pode haver um relacionamento ativo entre duas tabelas. O segundo, pela data de entrega, fica inativo e será ligado dentro de uma medida na Parte 6.

> 🔴 **Erro comum:** relacionar `fVendas` com `fMetas` diretamente. Tabelas fato não se relacionam entre si; elas conversam pelas dimensões em comum (`dLoja` e `dCalendario`).

> 🟢 **Checkpoint 3:** todas as setas apontam das dimensões para as fatos, com `1` no lado da dimensão e `*` no lado da fato.

---

## Parte 4: medidas básicas

Crie uma tabela só para guardar medidas: **Página Inicial → Inserir Dados → Carregar**, com o nome `Medidas`. Depois, **Nova medida** para cada uma:

```dax
faturamento = SUMX(fVendas, fVendas[valor_unitario] * fVendas[quantidade])

Lucro = SUMX(fVendas, (fVendas[valor_unitario] - fVendas[preco_custo]) * fVendas[quantidade])

margem_lucro = DIVIDE([Lucro], [faturamento])

qtde_pedidos = COUNTROWS(fVendas)

ticket_medio = AVERAGEX(fVendas, fVendas[quantidade] * fVendas[valor_unitario])

produtos_dist = DISTINCTCOUNT(fVendas[codigo_produto])

meta = SUM(fMetas[valor_meta])

dif (real-meta) = [faturamento] - [meta]

%Meta = DIVIDE([dif (real-meta)], [meta])
```

Formate `faturamento`, `Lucro`, `meta` e `ticket_medio` como moeda, e `margem_lucro` e `%Meta` como percentual.

> 🔵 **Por que `SUMX` e não `SUM`?** Não existe uma coluna "total" em `fVendas`. O `SUMX` percorre a tabela linha por linha, multiplica preço por quantidade e só então soma.

> 🔴 **Erro comum:** dividir com `/`. Se a meta for zero ou vazia, o visual mostra erro. O `DIVIDE` devolve vazio nesses casos.

> 🟢 **Checkpoint 4:** coloque as medidas em cartões, sem nenhum filtro, e compare:
>
> | Medida | Valor esperado |
> |---|---:|
> | `faturamento` | R$ 4.667.351,30 |
> | `Lucro` | R$ 2.004.411,30 |
> | `margem_lucro` | 42,9% |
> | `qtde_pedidos` | 3.000 |
> | `ticket_medio` | R$ 1.555,78 |
> | `meta` | R$ 4.675.442,62 |
> | `%Meta` | -0,2% |

---

## Parte 5: contexto de filtro com `CALCULATE`

```dax
QtdeProd_Maior2 = COUNTROWS(FILTER(fVendas, fVendas[quantidade] >= 3))

faturamento_lojaES = CALCULATE([faturamento], dLoja[nome_loja] = "Filial ES")

faturamento_2Lojas = CALCULATE([faturamento], dLoja[nome_loja] IN {"Filial ES", "Filial MG"})

faturamento_total = CALCULATE([faturamento], ALL(fVendas))

%Fat =
VAR v_faturamento = [faturamento]
VAR v_faturamentoAll = CALCULATE([faturamento], ALL(dLoja[nome_loja]))
RETURN DIVIDE(v_faturamento, v_faturamentoAll)
```

Monte uma tabela com `dLoja[nome_loja]`, `faturamento`, `faturamento_total` e `%Fat` para ver a diferença entre elas.

> 🟢 **Checkpoint 5:**
>
> | Loja | `faturamento` | `%Fat` |
> |---|---:|---:|
> | Matriz | R$ 1.123.153,60 | 24,1% |
> | Filial MG | R$ 1.395.685,60 | 29,9% |
> | Filial SP | R$ 973.079,80 | 20,8% |
> | Filial ES | R$ 1.175.432,30 | 25,2% |
>
> `faturamento_total` repete R$ 4.667.351,30 em todas as linhas e `QtdeProd_Maior2` dá 1.088.

---

## Parte 6: inteligência de tempo

```dax
faturamentoYTD = CALCULATE([faturamento], DATESYTD(dCalendario[Date]))

metaYTD = CALCULATE([meta], DATESYTD(dCalendario[Date]))

faturamentoLY = CALCULATE([faturamento], SAMEPERIODLASTYEAR(dCalendario[Date]))

Ano Atual vs Ano Anterior = [faturamento] - [faturamentoLY]

qtde_entregas =
CALCULATE(
    [qtde_pedidos],
    USERELATIONSHIP(fVendas[dt_entrega], dCalendario[Date])
)

tempo_medio_entrega (hs) =
AVERAGEX(
    fVendas,
    DATEDIFF(
        fVendas[dt_venda] + fVendas[hr_venda],
        fVendas[dt_entrega] + fVendas[hr_entrega],
        MINUTE
    ) / 60
)
```

> 🔴 **Erro comum:** usar a coluna de data da tabela fato (`fVendas[dt_venda]`) dentro de `DATESYTD` ou `SAMEPERIODLASTYEAR`. Essas funções precisam da coluna de data da tabela calendário.

> 🟢 **Checkpoint 6:** em uma tabela com `dCalendario[ano]`:
>
> | Ano | `faturamento` | `faturamentoLY` |
> |---|---:|---:|
> | 2019 | R$ 2.237.359,50 | (vazio) |
> | 2020 | R$ 2.429.991,80 | R$ 2.237.359,50 |
>
> `tempo_medio_entrega (hs)` fica em torno de 45 horas.

---

## Parte 7: montar as páginas

Antes de arrastar visuais, defina o que cada página responde.

**Página "Metas de vendas"**

| Visual | Campos |
|---|---|
| 3 cartões | `faturamento`, `meta`, `dif (real-meta)` |
| Tabela por loja | `nome_loja`, `faturamento`, `meta`, `%Meta` |
| Colunas agrupadas | `nome_mes` × `faturamento` e `meta` |
| Linhas | `nome_mes` × `faturamentoYTD` e `metaYTD` |
| Área ou colunas | `nome_mes` × `dif (real-meta)` |
| Segmentações | `ano` e `nome_loja` |

**Página "Visão geral de vendas"**

| Visual | Campos |
|---|---|
| Colunas por loja | `nome_loja` × `faturamento` |
| Linhas por hora | `int_hr_venda` × `qtde_pedidos` |
| Barras | `nome_loja` × `tempo_medio_entrega (hs)` |
| Linhas | `nome_mes` × `qtde_pedidos` e `qtde_entregas` |
| Colunas e linha | `Categoria` × `faturamento` (colunas) e `margem_lucro` (linha) |
| Tabela | vendedores, com `faturamento` e `ticket_medio` |

Cuidados de layout:

- Alinhe os visuais em uma grade e mantenha o mesmo espaçamento entre eles.
- Use uma cor principal e deixe as outras para destacar exceções (por exemplo, vermelho só para meta não batida).
- Dê a cada visual um título que diga o que ele mostra.
- Coloque os filtros sempre no mesmo lugar em todas as páginas.

> 🟢 **Checkpoint 7:** selecione "2020" e "Filial ES" nos filtros. Todos os visuais das duas páginas devem mudar. Se algum não mudar, revise os relacionamentos da Parte 3.

---

## Desafio final

A diretoria pediu uma novidade: **um ranking dos vendedores por faturamento, mostrando quem está no top 3.**

Tente sozinho. Se travar, abra as dicas na ordem.

<details>
<summary>Dica 1</summary>

Existe uma função DAX feita para ranking. Ela precisa de uma tabela para percorrer e de uma expressão para ordenar.
</details>

<details>
<summary>Dica 2</summary>

A função é `RANKX`. A tabela precisa ser a lista completa de vendedores, mesmo quando o visual está mostrando um só. Qual função remove o filtro de uma coluna?
</details>

<details>
<summary>Dica 3</summary>

A estrutura é `RANKX(ALL(dFuncionario[nome_funcionario]), [faturamento])`. Para marcar o top 3, compare o resultado com 3 dentro de um `IF`.
</details>

<details>
<summary>Solução</summary>

```dax
ranking_vendedor = RANKX(ALL(dFuncionario[nome_funcionario]), [faturamento])

top3 = IF([ranking_vendedor] <= 3, "Top 3", "Demais")
```

Sem filtros, o primeiro lugar é Aline Barreto, seguida de Fernanda Quintela e Marina Tavares.
</details>

---

## Para colocar no seu portfólio

1. Faça pelo menos uma mudança sua: o desafio do ranking, uma página de RH com `dFuncionario` ou um layout com a sua identidade visual.
2. Publique o `.pbix` e a base em um repositório seu, com prints das páginas.
3. No README, diga o que o dashboard responde, o que você mudou e cite este material como ponto de partida.
4. Use o [modelo de README de projeto](https://github.com/iasenai1111-ux/modelo-portfolio) como base.
