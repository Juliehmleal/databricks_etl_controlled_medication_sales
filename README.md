# Projeto de Análise de Venda de Medicamentos no Brasil

## Introdução

Este projeto de dados visa a construção de um pipeline ETL (Extract, Transform, Load) utilizando a **arquitetura de medalhão** para processar e analisar dados de venda de medicamentos controlados e antimicrobianos no Brasil.

A base de dados, proveniente do  Sistema Nacional de Gerenciamento de Produtos Controlados  [Base de dados SNGPC](https://dados.gov.br/dados/conjuntos-dados/venda-de-medicamentos-controlados-e-antimicrobianos---medicamentos-industrializados "Dados abertos gov br sngpc").   Disponibilizada publicamente pelo governo federal brasileiro. O objetivo principal é demonstrar a viabilidade de ingerir e tratar esses dados para suportar futuras aplicações de Business Intelligence (BI).

Os dados brutos são gerados a partir das movimentações de entrada e saída de medicamentos em farmácias e drogarias privadas do país, sendo a escrituração de responsabilidade do farmacêutico responsável técnico.

O projeto aborda um recorte temporal de **janeiro de 2014 a novembro de 2021**, pois a distribuição pública dos dados foi descontinuada após esse período.

## Objetivos do Projeto

O objetivo central é realizar o processo de ingestão, estruturação e modelagem dos dados para permitir o desenvolvimento de ferramentas analíticas.

* **Ingestão de Dados:** Tratar o grande volume de dados, que estão separados em arquivos CSV mensais, e transformá-los para o formato Delta na camada **Bronze**.
* **Estruturação e Modelagem Robusta:** Unificar todos os arquivos CSV em uma única tabela, tratando dados nulos, adaptando tipos de dados (já que todos vêm como string) e criando colunas para identificar o ano e o mês de cada registro.



## Arquitetura de Dados

O projeto segue a **arquitetura de medalhão** para organizar e processar os dados, com a governança centralizada pelo **Databricks Unity Catalog**. Essa abordagem garante a progressão da qualidade dos dados à medida que eles se movem através das camadas.

A estrutura lógica de armazenamento dos dados no Unity Catalog é a seguinte:

* **Catálogo `controlled_medication_data`:** Contêiner de alto nível do projeto, servindo como o ambiente principal.

* **Schemas (Camadas):** As camadas da arquitetura de medalhão são implementadas como schemas separados para isolar os dados em cada etapa do pipeline:

    * **`bronze_layer`:** Schema que armazena os dados brutos. Ele contém o **volume `raw_data`**, que serve como local de staging para os arquivos CSV baixados, e a tabela `tabelas_bronze`.

    * **`silver_layer`:** Schema onde os dados brutos são unificados, limpos e tratados. A tabela desta camada contém todos os dados do SNGPC consolidados, com tipos de dados corrigidos e valores nulos tratados, garantindo a confiabilidade dos dados.

    * **`gold_layer`:** Schema que armazena as tabelas finais, otimizadas para consumo de BI. Aqui é implementado um **star schema** e **views**, que são a fonte de dados para painéis e relatórios analíticos, como os criados no Power BI.



## Camada Bronze: Ingestão de Dados

A ingestão de dados foi realizada por meio de notebooks Python no Databricks, utilizando a biblioteca `requests` para fazer o download direto das URLs dos arquivos CSV.

* **Dados Principais (SNGPC):** O notebook `baixando_dados_brutos.py` foi responsável por baixar os arquivos mensais da base de dados de Venda de Medicamentos Controlados e Antimicrobianos. O processo percorre o período de 2014 a 2021 e salva cada arquivo CSV em um **Volume do Unity Catalog**, organizando-os em subdiretórios por ano.

* **Dados de Georreferenciamento:** O notebook `latitude_lingitude_dados.py` foi usado para obter dados de geolocalização dos municípios brasileiros, fundamentais para análises regionais. Os arquivos `estados.csv` e `municipios.csv` foram baixados de um repositório do GitHub e também armazenados nos volumes do Databricks.


## Camada Silver: Limpeza e Enriquecimento de Dados

O notebook `unificando_e_tratando_dados.py` é responsável pela transição dos dados da camada `bronze_layer` para a `silver_layer`. Ele executa o processo de unificação, limpeza e enriquecimento para criar a tabela principal de fatos do projeto. As principais etapas realizadas foram:

* **Unificação dos dados:** Os dataframes de cada ano (2014 a 2021) foram lidos individualmente e unidos em um único dataframe para criar uma tabela consolidada.
* **Tratamento de dados nulos:** Todas as linhas completamente vazias foram removidas. Além disso, os campos nulos de colunas importantes foram substituídos por valores padrão, como `-1` para colunas numéricas (`QTD_VENDIDA`, `IDADE`, `SEXO`) e `"N/A"` para colunas de texto.
* **Enriquecimento com dados geográficos:** O dataframe de vendas foi unido ao dataframe de municípios (disponível na camada `bronze_layer`) para adicionar informações geográficas, como a latitude e longitude, que serão essenciais para visualizações no databricks Dashboards.
* **Padronização de colunas:** Os nomes das colunas foram renomeados para o formato `snake_case`, melhorando a legibilidade e a consistência do conjunto de dados final.
* **Conversão de idade:** A idade dos compradores, que estava dividida em duas colunas (`IDADE` e `UNIDADE_IDADE`), foi unificada em uma única coluna (`idade_anos`) para simplificar a análise. O script também garante que idades menores que 1 ano sejam convertidas para 1, evitando valores inconsistentes.
* **Conversão de data:** As colunas `ANO_VENDA` e `MES_VENDA` foram combinadas para criar uma nova coluna `data_venda` no formato de data (`yyyy-MM-dd`), facilitando a análise temporal.

O resultado final é a tabela `vendas`, que é a principal fonte de dados para as próximas análises.


## Camada Ouro: Implementação do star schema
**Tabelas de Dimensão:**

* **`dim_data` (`dim_datas.py`):** Tabela que armazena informações únicas sobre cada data de venda, incluindo ano e mês.
* **`dim_comprador` (`dim_comprador.py`):** Contém informações sobre os compradores, como sexo e idade.
* **`dim_localizacao_venda` (`dim_localizacao.py`):** Armazena dados geográficos da venda, como UF, município, capital, região, latitude e longitude. Um tratamento especial foi realizado para o Distrito Federal.
* **`dim_prescritor` (`dim_prescritor.py`):** Contém informações sobre o profissional que prescreveu o medicamento, incluindo conselho, tipo de receituário e UF do conselho.
* **`dim_produto` (`dim_produto.py`):** Tabela com informações únicas sobre os medicamentos, incluindo o princípio ativo, a descrição da apresentação, a unidade de medida e o código CID10. Foi criada uma UDF (User-Defined Function) para classificar o formato do medicamento (ex.: "Comprimido", "Cápsula").

**Tabela Fato:**

* **`ft_vendas` (`fact_table.py`):** A tabela central do modelo, que armazena a quantidade de medicamentos vendidos (`qtd_vendida`) e faz a ligação com as tabelas de dimensão através de chaves estrangeiras. A criação da tabela de fatos envolveu a união (via left join) de todas as dimensões com a tabela da camada Silver.


## Orquestração de Pipeline

O pipeline ETL/ELT do projeto é orquestrado no Databricks Jobs, garantindo a execução programada e automatizada de todo o fluxo de dados. O processo foi dividido em duas pipelines principais, que rodam em sequência:

### Código da Pipeline

Todo Código da pipeline está na pasta arquitetura no arquivo pipeline, copie-o para dentro de um job e ele funcionará, este código pode ser replicado em seu próprio ambiente do Databricks para criar os jobs de orquestração dentro da pasta Arquitatura no arquivo PipeLine.YAML.
Importante ressaltar que deve-se alterar na parte de notebook_path para o seu email utilizado dentro do workspace databricks


## Genie e Governança com Unity Catalog

Para melhorar a governança dos dados e a experiência dos usuários, foram adicionadas descrições detalhadas a cada tabela de dimensão e à tabela de fatos diretamente no Unity Catalog. Essa documentação semântica é o que possibilita a funcionalidade do Genie, que permite aos usuários explorar os dados e realizar consultas usando linguagem natural, sem a necessidade de conhecimento técnico de SQL.

* **`dim_comprador`:** Esta tabela contém informações de clientes relacionadas ao sexo e à idade. Cada combinação dessas características recebe um identificador único. A tabela é utilizada como dimensão em um modelo star schema, permitindo análises demográficas dos clientes.
* **`dim_data`:** Esta tabela contém informações sobre as datas das vendas, incluindo data completa, ano e mês. Cada combinação dessas características possui um identificador único. A tabela é utilizada como dimensão em um modelo star schema, possibilitando análises temporais das vendas.
* **`dim_localizacao_venda`:** Esta tabela contém informações sobre a geolocalização das vendas, utilizada para suportar análises geográficas. Inclui os campos `uf_venda`, `nome_uf`, `municipio_venda`, `latitude_municipio`, `longitude_municipio`, `capital`, `latitude_uf`, `longitude_uf` e `regiao`. Cada combinação dessas características possui um identificador único. A tabela é utilizada como dimensão em um modelo star schema, possibilitando análises espaciais das vendas.
* **`dim_prescritor`:** Esta tabela contém informações sobre o prescritor da receita do medicamento vendido. Inclui os campos `conselho_prescritor`, `tipo_receituario` e `uf_conselho_prescritor`. Cada combinação dessas características possui um identificador único. A tabela é utilizada como dimensão em um modelo star schema, possibilitando análises relacionadas ao prescritor.
* **`dim_produto`:** Esta tabela contém informações sobre o medicamento vendido. Inclui os campos `princripio_ativo`, `descricao_apresentacao`, `unidade_medida`, `CID10`, `formato`. Cada combinação dessas características possui um identificador único. A tabela é utilizada como dimensão em um modelo star schema, possibilitando análises relacionadas aos medicamentos.
* **`ft_vendas`:** Esta tabela contém informações sobre as vendas de medicamentos. Inclui o campo `qtd_vendida`, juntamente com os identificadores de cada tabela dimensão no modelo star schema. Esta tabela desempenha o papel de fato, centralizando os dados transacionais que permitem análises de volume de vendas, distribuição geográfica, período de ocorrência e demais métricas relacionadas.
![chrome_0nTikivXWL](https://github.com/user-attachments/assets/d7288af4-0426-44e1-a720-ee86acaa8727)


## Análises de Business Intelligence (BI)
Com os dados tratados e enriquecidos nas camadas Silver e Gold, foram desenvolvidos os seguintes painéis de BI para análise:
* **Vendas por Estado:** Um mapa que exibe as vendas totais por estado, utilizando as colunas de latitude e longitude para uma visualização geográfica precisa.
* **Ranking de Medicamentos Mais Vendidos:**  Um dashboard que classifica os princípios ativos por volume de vendas, destacando os medicamentos mais populares no período.
  ![chrome_nTGpnnHe0A](https://github.com/user-attachments/assets/b609a501-3614-4dc3-8cc7-654d828c5d9b)


* **Vendas por Cidade:** Um painel que permite analisar o total de vendas por município, permitindo uma visão detalhada do consumo em nível local.
* **Histórico Temporal de Vendas:** Uma análise que mostra o histórico de vendas por princípio ativo ao longo do tempo.
   ![chrome_8zxMDcESyB](https://github.com/user-attachments/assets/51f0fe41-b078-4d50-8764-c5980e550e51)
  
* **Relatório Geral:** Um dashboard abrangente que consolida as principais métricas e permite a aplicação de filtros em todos os campos, como período, cidade e estado, para análises personalizadas.
![chrome_Kfl1E5xZCX](https://github.com/user-attachments/assets/8b5abf48-7332-4b0a-8c0a-7b0a239d2801)


## Tecnologias Utilizadas

O projeto foi inteiramente desenvolvido no **Databricks**, que atua como um ambiente unificado para o ciclo de vida dos dados, do ETL à análise.

* **Plataforma de Desenvolvimento:** Databricks
* **Linguagens de Programação:** Python e SQL
* **Framework de Processamento:** Apache Spark, nativamente integrado ao Databricks
* **Controle de Versão:** Git/GitHub, para gerenciar o versionamento do código-fonte

Todo o código-fonte do projeto, incluindo scripts de ingestão e transformação, será disponibilizado publicamente neste repositório do GitHub para promover a transparência e permitir que outros possam estudar a metodologia.
