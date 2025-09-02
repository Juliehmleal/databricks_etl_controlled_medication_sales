# Projeto de Análise de Venda de Medicamentos no Brasil

## Introdução

Este projeto de dados visa a construção de um pipeline ETL (Extract, Transform, Load) utilizando a **arquitetura de medalhão** para processar e analisar dados de venda de medicamentos controlados e antimicrobianos no Brasil.

A base de dados, proveniente do Sistema Nacional de Gerenciamento de Produtos Controlados (SNGPC), é disponibilizada publicamente pelo governo federal brasileiro. O objetivo principal é demonstrar a viabilidade de ingerir e tratar esses dados para suportar futuras aplicações de Business Intelligence (BI).

[Base de dados SNGPC](https://dados.gov.br/dados/conjuntos-dados/venda-de-medicamentos-controlados-e-antimicrobianos---medicamentos-industrializados "Dados abertos gov br sngpc").

Os dados brutos são gerados a partir das movimentações de entrada e saída de medicamentos em farmácias e drogarias privadas do país, sendo a escrituração de responsabilidade do farmacêutico responsável técnico.

O projeto aborda um recorte temporal de **janeiro de 2014 a novembro de 2021**, pois a distribuição pública dos dados foi descontinuada após esse período.

## Arquitetura do Projeto

O projeto adota a **arquitetura de medalhão**, que organiza os dados em camadas para garantir a qualidade e o enriquecimento progressivo das informações.

* **Camada Bronze:** Recebe os dados brutos (raw data) diretamente da fonte, sem transformações iniciais, mantendo a integridade dos dados originais.
* **Camada Silver:** Os dados da camada Bronze passam por limpeza e normalização, incluindo padronização de formatos, remoção de duplicatas e tratamento de valores nulos.
* **Camada Gold:** A fonte de dados para aplicações de análise e consumo. Os dados são enriquecidos, agregados e otimizados para ferramentas de BI.

Todo o fluxo de dados, da ingestão inicial ao enriquecimento final, é executado inteiramente na plataforma **Databricks**, utlizando a versão Free Edition.

## Objetivos do Projeto

O objetivo central é realizar o processo de ingestão, estruturação e modelagem dos dados para permitir o desenvolvimento de ferramentas analíticas.

### Egenharia de dados (ETL)

* **Ingestão de Dados:** Tratar o grande volume de dados, que estão separados em arquivos CSV mensais, e transformá-los para o formato Delta na camada **Bronze**.
* **Estruturação e Modelagem Robusta:** Unificar todos os arquivos CSV em uma única tabela, tratando dados nulos, adaptando tipos de dados (já que todos vêm como string) e criando colunas para identificar o ano e o mês de cada registro.
* **Modelagem Star Schema:** Visando otimizar as buscas feitas na camada ouro será usada a abordagem de star schema.

## Análises de Business Intelligence (BI)

Com os dados tratados e enriquecidos nas camadas **Silver** e **Gold**, será possível criar painéis de BI para as seguintes análises:

* **Relatório Geral com Filtros:** Análise por meio dos filtros podendo ver dados conforme o desejado
* **Graficos de vendas de medicamentos por Estado/cidade:** Análise do numero de vendas total de medicamentos por cidade.
* **Vendas por Região e Período:** Análise do volume de vendas por Unidade Federativa (UF), Município, Ano e Mês. Indicadores como o ranking dos 10 estados ou municípios com maior volume de consumo serão desenvolvidos. Essa análise é viável pois os dados contêm informações completas de geolocalização e tempo.
* **Ranking de Princípios Ativos:** Identificação e classificação dos princípios ativos mais vendidos. 
* **Mapa de vendas total no Brasil:** Gráfico em formato de mapa com o total vendas por estado, mostrando visualmente as vendas por estado por meio de granularidade
* **Gráfico temporal de histórico de vendas por princípio ativo:** Ánalises das alterações de venda de um princípio ativo ao longo dos anos


## Tecnologias Utilizadas

O projeto foi inteiramente desenvolvido no **Databricks**, que atua como um ambiente unificado para o ciclo de vida dos dados, do ETL à análise.

* **Plataforma de Desenvolvimento:** Databricks
* **Linguagens de Programação:** Python e SQL
* **Framework de Processamento:** Apache Spark, nativamente integrado ao Databricks
* **Controle de Versão:** Git/GitHub, para gerenciar o versionamento do código-fonte
