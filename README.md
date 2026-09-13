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
| Valor total registrado | Soma de `vl_preco_total` | Soma bruta, sem segmentação — base para os filtros do dashboard |
| Quantidade total de itens comprados | Soma de `qt_medicamento` | Soma bruta entre unidades de medida distintas (comprimido, mililitro, grama etc.) — não deve ser interpretada como itens fisicamente equivalentes entre si |
| Número de registros de compra | Contagem de linhas | Após aplicação dos filtros |
| Instituições compradoras | Contagem distinta de `cnpj_instituicao` | CNPJ usado em vez do nome para evitar inflação por variações de grafia |
| Fornecedores | Contagem distinta de `cnpj_fornecedor` | Mesma lógica acima |
| Preço unitário médio ponderado | `SUM(vl_preco_total) / SUM(qt_medicamento)` | **Nunca calculado como média de `vl_preco_unitario`** — essa abordagem distorceria o resultado ao dar peso igual a compras de tamanhos muito diferentes. Deve ser interpretado com cautela quando os filtros incluírem produtos, unidades de fornecimento ou apresentações distintas. |

## Limitações identificadas

- Um pequeno grupo de registros (31 de 367.003, 0,008% das linhas) concentra 12,3% da quantidade total comprada — em sua maioria medicamentos de alto consumo no SUS, consistentes com compras centralizadas de grande escala. Não há evidência suficiente de erro de digitação para justificar exclusão, mas o preço unitário médio ponderado é sensível a essa concentração (variação de 11% com/sem esses registros).
- 90 registros (0,11%) não possuem classificação de PDM/Grupo/Classe, concentrados 100% no tipo de compra `ADMINISTRATIVA`.
- `dt_insercao` pode ocorrer anos após `dt_compra` (até 6 anos de defasagem observada); toda análise temporal do dashboard usa `dt_compra`.
- O arquivo de 2026 é parcial (cobertura até meados do ano).

## Sprint 5 — Investigação de Qualidade de Dados

Durante a análise de dispersão de preços (Pergunta de Negócio 5), identificamos que
aproximadamente 57% do Valor Total Registrado original (R$ 115.063.593.346,89) provinha
de registros com preço unitário anormal — 50x ou mais acima da mediana do próprio item
(mesmo código CATMAT).

**Diagnóstico:** ao comparar com os CSVs brutos da fonte, confirmamos que o erro já está
presente nos dados originais publicados pelo Ministério da Saúde, não foi introduzido no
nosso pipeline de preparação. O padrão identificado (98,9% dos casos concentrados em
fatores de ~100x ou ~1.000x) é consistente com erro sistemático de casa decimal na
publicação da fonte.

**Correção aplicada:** para cada registro com razão de preço/mediana acima de 50x,
dividimos o preço unitário pela potência de 10 correspondente à ordem de grandeza do erro.
Os valores originais foram preservados nas colunas `vl_preco_unitario_bruto` e
`vl_preco_total_bruto`, e uma coluna `flag_preco_corrigido` sinaliza quais dos 367.003
registros foram ajustados (1.812 registros, 0,49% do total em contagem — mas 57% em valor).

**Valor Total corrigido: R$ 49.305.949.104,98** (usado no dashboard a partir desta sprint).

Metodologia completa disponível em `data/scripts/analise_precos_sprint5.ipynb`.

## Pergunta 6 — Modalidade de Compra e Relação com o Preço

O campo `modalidade` (distinto de `tp_compra`, que separa apenas natureza
Administrativa/Judicial) identifica a modalidade de licitação de cada registro de compra.

| Modalidade | Registros | Valor Total | Preço Médio Ponderado | % do Valor Total |
|---|---:|---:|---:|---:|
| Pregão | 332.087 | R$ 43.843.040.000 | R$ 0,7553 | 88,92% |
| Registro de Preços | 18.007 | R$ 3.742.638.000 | R$ 0,7514 | 7,59% |
| Dispensa de Licitação | 13.599 | R$ 1.634.571.000 | R$ 0,9441 | 3,32% |
| Inexigibilidade de Licitação | 417 | R$ 66.140.540 | R$ 4,8392 | 0,13% |
| Tomada de Preços | 1.864 | R$ 10.591.720 | R$ 0,7401 | 0,02% |
| Concorrência | 710 | R$ 3.566.332 | R$ 0,5068 | 0,01% |
| Concurso | 194 | R$ 2.637.446 | R$ 0,4997 | 0,01% |
| Leilão | 54 | R$ 2.002.841 | R$ 0,2179 | 0,00% |
| Convite | 70 | R$ 749.631 | R$ 7,1502 | 0,00% |
| Diálogo Competitivo | 1 | R$ 15.228 | R$ 1,4100 | 0,00% |

**Interpretação:**

- **Pregão domina amplamente** (88,9% do valor total), o que é esperado — é a
  modalidade padrão para compras públicas competitivas no Brasil.
- **Dispensa de Licitação tem preço unitário ~25% maior** que Pregão (R$ 0,9441 vs.
  R$ 0,7553), consistente com a expectativa de que processos menos competitivos
  resultam em preços menos vantajosos para o poder público.
- **Inexigibilidade de Licitação aparece com preço médio ~6,4x maior** que o Pregão.
  Isso não deve ser interpretado como irregularidade: essa modalidade é usada
  legalmente quando existe apenas um fornecedor possível (ex: medicamento
  patenteado/exclusivo), então um preço mais alto pode ser genuíno e justificado
  pela ausência de concorrência real, não necessariamente sobrepreço.

## Achado Adicional — Judicialização da Saúde

Durante a exploração dos dados, identificamos que o campo `tp_compra` (natureza da
compra, distinto de `modalidade`) revela que **7,15% do valor total de compras**
(aproximadamente R$ 3,5 bilhões, após correção de preços da Sprint 5) é movimentado
por decisão **Judicial**, contra 92,85% por via **Administrativa** normal. Esse é um
indicador relevante de judicialização da saúde no Brasil, tema amplamente debatido
em políticas públicas de saúde, embora não fizesse parte das 6 perguntas de negócio
originais do projeto.


## Pergunta 3 — Validação: Agrupamento por CATMAT vs. Descrição

Antes de consolidar o Top 10 de Medicamentos, validamos se agrupar por código CATMAT
(`co_catmat`) produzia resultado diferente de agrupar pela descrição textual (`ds_item`),
já usada no dashboard. Confirmamos que **nenhum CATMAT possui mais de uma descrição
distinta** em todo o dataset (367.003 registros), e o Top 10 por ambos os métodos é
idêntico. Isso valida que o gráfico "Top 10 Medicamentos" do dashboard responde
corretamente à Pergunta 3, sem risco de fragmentação de valor por inconsistência de
cadastro.


## Pergunta 4 — Fornecedores e Fabricantes

Além do Top 10 Fornecedores (distribuidores que efetivamente vendem ao poder público,
já no dashboard), analisamos separadamente o Top 10 Fabricantes (indústria que produz
o medicamento/dispositivo):

| Fabricante | Valor Total | Registros | % do Valor Total |
|---|---:|---:|---:|
| Novartis Biociências SA | R$ 3.919.206.000 | 4.266 | 7,95% |
| Janssen-Cilag Farmacêutica LTDA | R$ 2.079.847.000 | 865 | 4,22% |
| AstraZeneca do Brasil LTDA. | R$ 1.857.312.000 | 1.622 | 3,77% |
| Sanofi Medley Farmacêutica LTDA | R$ 1.585.869.000 | 4.013 | 3,22% |
| Aché Laboratórios Farmacêuticos SA | R$ 1.576.320.000 | 4.281 | 3,20% |
| Cristália Produtos Químicos Farmacêuticos LTDA | R$ 1.538.333.000 | 22.397 | 3,12% |
| EMS S/A | R$ 1.519.386.000 | 18.438 | 3,08% |
| Boehringer Ingelheim do Brasil Química e Farmacêutica LTDA. | R$ 1.486.342.000 | 2.050 | 3,01% |
| Prati, Donaduzzi & Cia LTDA | R$ 1.463.806.000 | 24.260 | 2,97% |
| GlaxoSmithKline Brasil LTDA | R$ 1.391.008.000 | 1.609 | 2,82% |

**Interpretação:**

- Os 10 maiores fabricantes concentram **37,35% do valor total**, uma concentração
  moderada — menos extrema que a observada entre instituições e fornecedores.
- **Novartis lidera em valor** (7,95%) com relativamente poucos registros (4.266),
  sugerindo produtos de ticket unitário alto (biológicos/especialidades).
- **Cristália e Prati-Donaduzzi** aparecem com 5-6x mais registros que a Novartis mas
  valor total similar ou menor, indicando perfil de fabricantes de medicamentos
  genéricos de alto volume e menor preço unitário — perfil de negócio distinto dentro
  do mesmo ranking.
  
