# Projeto BPS — Banco de Preços em Saúde

**Aluno:** Waldinei Lameira Rosa
**Turma:** Visualização de Dados e Business Intelligence — Turma 2 (T2)
**Módulo:** Módulo 2 — Mini-Projeto Avaliativo (Semana 06/07)

## Perguntas de negócio

Este dashboard foi desenvolvido para responder às seguintes questões:

1. Como o valor total de compras registradas no BPS evoluiu ano a ano (2020–2026)?
2. Quais estados (UF) e instituições concentram o maior volume financeiro de compras?
3. Quais medicamentos e dispositivos médicos (identificados por código CATMAT) têm maior valor total adquirido?
4. Quais fornecedores e fabricantes têm maior participação nos registros de compra?
5. Para um mesmo item (mesmo código CATMAT), como o preço unitário varia entre instituições, estados e períodos?
6. Quais modalidades de compra (licitação, pregão, compra direta, entre outras) são mais utilizadas, e como isso se relaciona com o preço pago?

## Como reproduzir o dataset consolidado

O arquivo `BPS_20_26_Waldinei.csv` não está versionado neste repositório por exceder o limite prático de tamanho do GitHub (~200MB). Para gerá-lo localmente:

1. Baixe os arquivos CSV de 2020 a 2026 em https://dadosabertos.saude.gov.br/dataset/bps e salve-os em `data/raw/`, nomeados como `2020.csv`, `2021.csv`, ..., `2026.csv`.
2. Instale as dependências: `pip install pandas`.
3. Execute o notebook `scripts/preparacao_bps.ipynb` do início ao fim (Run All).
4. O arquivo consolidado será gerado em `data/processed/BPS_20_26_Waldinei.csv`.

## Tratamentos e transformações realizadas

- **Datas** (`dt_compra`, `dt_insercao`): convertidas de texto para datetime. Toda análise temporal do dashboard usa `dt_compra` como referência — `dt_insercao` pode ocorrer anos depois da compra e serve apenas como metadado administrativo.
- **CNPJs**: convertidos para texto com 14 dígitos (zeros à esquerda restaurados, perdidos na leitura original como número).
- **Códigos identificadores** (`co_pdm`, `co_grupo`, `co_classe`, `registro_anvisa`): convertidos para inteiro anulável (`Int64`), eliminando o sufixo `.0` indevido causado por valores nulos.
- **Nulos**: mantidos sem inferência nos campos onde são esperados (ex.: `nu_ata`, presente em apenas 0,66% dos registros, já que a maioria das compras não utiliza ata de registro de preços). Uma exceção investigada: 90 registros sem classificação de PDM/Grupo/Classe são 100% do tipo de compra `ADMINISTRATIVA`, sugerindo que esse tipo de aquisição não passa pela classificação padrão do BPS.
- **Duplicadas**: nenhuma encontrada em nenhum dos 7 anos, nem no dataset consolidado.
- **Consolidação**: os 7 arquivos foram concatenados por empilhamento simples (`pd.concat`), já que a estrutura de colunas é idêntica em todos os anos (ver `docs/discrepancias_anos.md`). Validado que a soma de linhas por ano bate com o total consolidado (367.003 linhas), sem perdas, e que todas as linhas têm `ano_compra` coincidindo com o ano do arquivo de origem.

## Resultado da consolidação

- **367.003 linhas**, 37 colunas (36 originais + `ano_arquivo_origem` para rastreabilidade)
- Distribuição por ano: 2020 (84.919) · 2021 (85.007) · 2022 (89.534) · 2023 (33.785) · 2024 (28.745) · 2025 (34.010) · 2026 (11.003 — ano parcial)

## KPIs e métricas do dashboard

| KPI | Fórmula | Observação |
|---|---|---|
| Valor total registrado | Soma de `vl_preco_total` | — |
| Quantidade total de itens comprados | Soma de `qt_medicamento` | — |
| Número de registros de compra | Contagem de linhas | Após aplicação dos filtros |
| Instituições compradoras | Contagem distinta de `cnpj_instituicao` | CNPJ usado em vez do nome para evitar inflação por variações de grafia |
| Fornecedores | Contagem distinta de `cnpj_fornecedor` | Mesma lógica acima |
| Preço unitário médio ponderado | `SUM(vl_preco_total) / SUM(qt_medicamento)` | **Nunca calculado como média de `vl_preco_unitario`** — essa abordagem distorceria o resultado ao dar peso igual a compras de tamanhos muito diferentes. Deve ser interpretado com cautela quando os filtros incluírem produtos, unidades de fornecimento ou apresentações distintas. |

## Limitações identificadas

- Um pequeno grupo de registros (31 de 367.003, 0,008% das linhas) concentra 12,3% da quantidade total comprada — em sua maioria medicamentos de alto consumo no SUS, consistentes com compras centralizadas de grande escala. Não há evidência suficiente de erro de digitação para justificar exclusão, mas o preço unitário médio ponderado é sensível a essa concentração (variação de 11% com/sem esses registros).
- 90 registros (0,11%) não possuem classificação de PDM/Grupo/Classe, concentrados 100% no tipo de compra `ADMINISTRATIVA`.
- `dt_insercao` pode ocorrer anos após `dt_compra` (até 6 anos de defasagem observada); toda análise temporal do dashboard usa `dt_compra`.
- O arquivo de 2026 é parcial (cobertura até meados do ano).

