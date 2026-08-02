# 🐍 Análise Exploratória e Limpeza de Dados de Vendas Globais

<p align="center">
  <img src="https://img.shields.io/badge/Python-3776AB?style=for-the-badge&logo=python&logoColor=white" alt="Python"/>
  <img src="https://img.shields.io/badge/Pandas-2C2D72?style=for-the-badge&logo=pandas&logoColor=white" alt="Pandas"/>
  <img src="https://img.shields.io/badge/Jupyter-F37626.svg?&style=for-the-badge&logo=Jupyter&logoColor=white" alt="Jupyter"/>
</p>

> **💡 Resumo do Projeto**
> Este repositório contém o pipeline completo de ingestão, limpeza, unificação e Análise Exploratória de Dados (EDA) de uma operação de e-commerce e varejo global (focada em B2B/Atacado). O objetivo principal foi diagnosticar a qualidade dos dados, realizar tratamentos de inconsistências sistêmicas (falsos nulos, tipagem e ruídos) e extrair inteligência de negócios relacionando volume de vendas e rentabilidade.

---

## 📌 Principais Métricas de Negócio
Após a consolidação e limpeza da base, a operação registrou o seguinte panorama:

| Métrica | Resultado |
| :--- | :--- |
| **Total de Pedidos** | 1.328 |
| **Unidades Vendidas** | 6.57 Milhões |
| **Receita Total** | $ 1.70 Bilhões |
| **Lucro Total** | $ 501.4 Milhões |
| **Ticket Médio** | $ 1.28 Milhão |
| **Margem Média** | 29,46% |
| **Alcance Global** | 46 Países atendidos |

---

## 🛠️ 1. Diagnóstico e Limpeza dos Dados (Data Cleaning)
A base original estava fragmentada em três tabelas (`events.csv`, `products.csv`, `countries.csv`). Antes do cruzamento dos dados, apliquei uma etapa rigorosa de validação de qualidade:

* **Tratamento do "Falso Nulo" (Namíbia):** Identificou-se que o código ISO *alpha-2* da Namíbia (`"NA"`) estava sendo interpretado pelo Pandas como um valor ausente (`NaN`). A correção foi aplicada garantindo a integridade geográfica.
* **Limpeza Estrutural:** Exclusão cirúrgica de apenas 2 registros (0,15% da base) por ausência crítica na métrica de vendas (`Units Sold`), priorizando a exatidão financeira no lugar de imputações por mediana.
* **Gestão de Dados Ausentes:** 82 registros sem país de origem mapeado receberam a tag `"Unknown"`, preservando suas expressivas métricas financeiras.
* **Transformações e Casting:** Conversão de strings para `datetime` (`Order Date`, `Ship Date`), casting para `int` em unidades, e padronização de nomenclatura (espaços extras e capitalização) em colunas categóricas (`Sales Channel`, `Order Priority`).

<details>
  <summary><b>💻 Clique para ver o snippet de código (Limpeza em Pandas)</b></summary>

```python
# 1. Corrige o falso nulo de Namibia no dataframe de países
df_countries.loc[df_countries['name'] == 'Namibia', 'alpha-2'] = 'NA'

# 2. Remove as 2 linhas com Units Sold ausente
df_events = df_events.dropna(subset=['Units Sold'])

# 3. Preenche os 82 valores ausentes de Country Code com "Unknown"
df_events['Country Code'] = df_events['Country Code'].fillna('Unknown')

# 4. Converte Order Date e Ship Date para datetime
df_events['Order Date'] = pd.to_datetime(df_events['Order Date'])
df_events['Ship Date'] = pd.to_datetime(df_events['Ship Date'])

# 5. Padroniza Sales Channel e Order Priority
df_events['Sales Channel'] = df_events['Sales Channel'].str.strip().str.capitalize()
df_events['Order Priority'] = df_events['Order Priority'].str.strip()
```
</details>

---

## 🔗 2. Relacionamento e Feature Engineering
As tabelas foram unificadas via `Merge` (`LEFT JOIN`), utilizando `Product ID` e `Country Code` como chaves. 

* **Otimização do DataFrame Final:** Colunas de chaves redundantes e identificadores alfanuméricos foram descartadas. O fluxo temporal foi validado logicamente (nenhum *Ship Date* precedendo *Order Date*).
* **Engenharia de Recursos (Feature Engineering):** Foram criadas as features financeiras fundamentais baseadas na volumetria de vendas e custos unitários (`Revenue`, `Cost`, `Profit`).

<details>
  <summary><b>💻 Clique para ver o snippet de código (Merge e Features)</b></summary>

```python
# Merge das tabelas mantendo a granularidade das transações
df_final = df_events.merge(df_products, how='left', left_on='Product ID', right_on='id')
df_final = df_final.merge(df_countries, how='left', left_on='Country Code', right_on='alpha-3')

# Limpeza de colunas redundantes e renomeação padronizada
df_final['name'] = df_final['name'].fillna('Unknown')
df_final = df_final.drop(columns=['id', 'alpha-2', 'alpha-3', 'Country Code', 'Product ID'])
df_final = df_final.rename(columns={
    'name': 'Country', 'item_type': 'Product Category', 
    'region': 'Region', 'sub-region': 'Sub-Region'
})

# Criação das novas variáveis financeiras
df_final['Revenue'] = df_final['Unit Price'] * df_final['Units Sold']
df_final['Cost'] = df_final['Unit Cost'] * df_final['Units Sold']
df_final['Profit'] = df_final['Revenue'] - df_final['Cost']
```
</details>

---

## 📊 3. Análise Exploratória (EDA) e Insights

A exploração visual e agrupamento dos dados trouxe respostas claras sobre eficiência operacional através da agregação das métricas recém-calculadas:

### O Paradigma do Volume vs. Rentabilidade
A receita bruta não conta a história completa da empresa. Identificamos discrepâncias críticas entre o faturamento e o dinheiro que efetivamente entra no caixa:
* **Falso Ouro:** A categoria `Office Supplies` é a campeã em receita bruta ($402,2M), porém apresenta a segunda pior margem de lucro de todo o portfólio (19,4%). Exige esforço logístico alto para um retorno percentual baixo.
* **Categoria Premium:** `Clothes` sustenta a maior margem de lucro absoluta do negócio (**67,2%**). É a categoria mais eficiente.
* **O Motor da Empresa:** `Cosmetics` é a linha de frente do negócio. Com uma margem atrativa de 39,7%, lidera a base em lucro absoluto gerando $92,7 Milhões.

<details>
  <summary><b>💻 Clique para ver o snippet de código (Agregação EDA)</b></summary>

```python
# Agrupamento das métricas financeiras por Categoria de Produto
df_categoria = df_final.groupby('Product Category').agg({
    'Revenue': 'sum',
    'Cost': 'sum',
    'Profit': 'sum',
    'Units Sold': 'sum'
}).reset_index()

# Cálculo da Margem de Lucro (%) para ranqueamento
df_categoria['Profit Margin (%)'] = (df_categoria['Profit'] / df_categoria['Revenue']) * 100
df_categoria_sorted = df_categoria.sort_values('Profit', ascending=False)
```
</details>

<br>

> ⚠️ **Nota Visual:** Os gráficos interativos gerados com as bibliotecas `Matplotlib` e `Seaborn` demonstrando o desempenho cruzado podem ser visualizados diretamente no Notebook disponível neste repositório.

---

## 📁 Estrutura do Repositório

```text
analise-vendas-globais/
│
├── data/
│   ├── countries.csv             # Catálogo de países
│   ├── events.csv                # Registros transacionais brutos
│   └── products.csv              # Catálogo de produtos
│
├── notebooks/
│   └── eda_vendas_globais.ipynb  # Código fonte da análise completa
│
├── assets/                       # Gráficos e imagens gerados na análise
│
├── requirements.txt              # Dependências do projeto
└── README.md                     # Apresentação do projeto
```
