# 🚀 Laboratório Prático: Data Lake & Analytics na AWS — Lab 7

Este laboratório demonstra o fluxo de ingestão, limpeza, catalogação e análise de dados utilizando serviços da AWS.

## 🏗️ Arquitetura

```text
[Arquivo CSV]
      |
      v
[Amazon S3 - Raw]
      |
      v
[AWS Glue DataBrew]
      |
      v
[Amazon S3 - Curated / Parquet]
      |
      v
[AWS Glue Crawler]
      |
      v
[Glue Data Catalog]
      |
      v
[Amazon Athena - SQL]
```

**Serviços opcionais, apenas conceituais:** Amazon EMR e Amazon Redshift Spectrum.

## 📋 1. Pré-requisitos

- Acesso ao AWS Management Console (AWS Learner Lab).
- Permissões para utilizar S3, CloudFormation, Glue, DataBrew e Athena.
- Editor de texto para criar o arquivo CSV.

> **Atenção aos custos:** utilize apenas os recursos necessários, acompanhe as cobranças e exclua os recursos ao terminar. Não crie clusters EMR nem ambientes Redshift neste laboratório.

## 📁 2. Preparação dos dados

Crie o arquivo `vendas_2026.csv` com o seguinte conteúdo:

```csv
id_venda,data_venda,cliente,valor,categoria
1001,2026-01-15,José Santana,250.00,Eletronicos
1002,2026-01-16,Maria Silva,,Eletronicos
1003,2026-01-16,Carlos Souza,120.50,Vestuario
1004,2026/01/17,Ana Costa,450.00,Casa
1005,2026-01-18,Paulo Santos,89.90,Vestuario
1006,,Lucas Lima,1200.00,Eletronicos
```

O arquivo contém valores ausentes e uma data em formato diferente, que serão tratados durante a limpeza.

## ⚙️ 3. Provisionamento com AWS CloudFormation

O CloudFormation será utilizado para criar o bucket S3 e o banco de dados do Glue Catalog.

Como o laboratório disponibiliza somente a role `LabRole`, o template abaixo não tenta criar uma nova role IAM.

Crie o arquivo `template.yaml`:

```yaml
AWSTemplateFormatVersion: '2010-09-09'

Description: Infraestrutura base para o laboratorio Lab 7

Resources:
  DataLakeLab7Bucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub 'datalake-lab7-${AWS::AccountId}-${AWS::Region}'
      PublicAccessBlockConfiguration:
        BlockPublicAcls: true
        BlockPublicPolicy: true
        IgnorePublicAcls: true
        RestrictPublicBuckets: true

  Lab7GlueDatabase:
    Type: AWS::Glue::Database
    Properties:
      CatalogId: !Ref AWS::AccountId
      DatabaseInput:
        Name: db_lab7

Outputs:
  BucketName:
    Description: Nome do bucket S3 do Lab 7
    Value: !Ref DataLakeLab7Bucket

  GlueDatabaseName:
    Description: Nome do banco de dados do Glue Catalog
    Value: !Ref Lab7GlueDatabase
```

### Criar a stack

1. Acesse **CloudFormation** no console da AWS.
2. Selecione **Create stack → With new resources**.
3. Envie o arquivo `template.yaml`.
4. Nomeie a stack como `datalake-lab7`.
5. Avance e envie a solicitação.
6. Após a criação, acesse **Outputs** e copie o valor de `BucketName`.

> O template não cria permissões IAM. Nos serviços seguintes, selecione `LabRole` quando ela for aceita e tiver as permissões necessárias. Se alguma operação for bloqueada, não tente contornar as restrições do laboratório.

## ☁️ 4. Ingestão no Amazon S3

1. Abra o **Amazon S3** e acesse o bucket criado.
2. Crie a pasta `raw/vendas/`.
3. Entre na pasta e envie o arquivo `vendas_2026.csv`.

O caminho será:

```text
s3://<nome-do-bucket>/raw/vendas/vendas_2026.csv
```

Substitua `<nome-do-bucket>` pelo nome obtido no CloudFormation.

## 🧹 5. Limpeza com AWS Glue DataBrew

### Criar o dataset

1. Abra o **AWS Glue DataBrew**.
2. Acesse **Datasets → Connect new dataset**.
3. Configure:

   - **Dataset name:** `ds_lab7_vendas_raw`
   - **Source:** Amazon S3
   - **S3 location:** `s3://<nome-do-bucket>/raw/vendas/vendas_2026.csv`

4. Salve o dataset e execute a perfilagem dos dados (*Run data profile*).

### Criar o projeto

1. Acesse **Projects → Create project**.
2. Configure:

   - **Project name:** `proj-lab7-limpeza-vendas`
   - **Dataset:** `ds_lab7_vendas_raw`
   - **Role:** `LabRole`, se disponível e autorizada.

3. Na interface visual, aplique as transformações:

   - Remova registros com `valor` ausente.
   - Remova registros com `data_venda` ausente.
   - Padronize as datas para `YYYY-MM-DD`, tratando o formato `2026/01/17`.

### Criar o job

Crie um job a partir da receita:

| Campo | Valor |
|---|---|
| Job name | `job-lab7-clean-vendas` |
| Output format | Parquet |
| Output location | `s3://<nome-do-bucket>/curated/vendas/` |

Publique a receita, se solicitado, e execute o job.

## 🗂️ 6. Catalogação com AWS Glue Crawler

1. Abra o **AWS Glue → Crawlers**.
2. Crie um crawler chamado `crawler-lab7-vendas-curated`.
3. Configure:

   - **Data source:** Amazon S3
   - **S3 path:** `s3://<nome-do-bucket>/curated/vendas/`
   - **IAM role:** `LabRole`, se disponível e autorizada.
   - **Target database:** `db_lab7`

4. Conclua a criação e execute o crawler.
5. Após a execução, acesse **Data Catalog → Tables** e localize a tabela criada.

Utilize o nome efetivamente registrado no catálogo nas consultas seguintes. Neste roteiro, será utilizado `vendas`.

## 📊 7. Consultas com Amazon Athena

### Configurar o local de resultados

1. Abra o **Amazon Athena**.
2. Acesse as configurações do editor (*Settings*).
3. Configure o local de saída:

```text
s3://<nome-do-bucket>/athena-results/
```

4. Selecione o banco de dados `db_lab7`.

### Consulta por categoria

Execute a consulta:

```sql
SELECT
    categoria,
    COUNT(id_venda) AS qtd_vendas,
    ROUND(SUM(valor), 2) AS faturamento_total
FROM "db_lab7"."vendas"
GROUP BY categoria
ORDER BY faturamento_total DESC;
```

### Consultas adicionais

**Quantidade de registros:**

```sql
SELECT COUNT(*) AS total_registros
FROM "db_lab7"."vendas";
```

**Faturamento total:**

```sql
SELECT ROUND(SUM(valor), 2) AS faturamento_total
FROM "db_lab7"."vendas";
```

Considerando as transformações propostas, o conjunto deverá conter quatro registros, com faturamento total de `910.40`.

## 🔬 8. Cenários opcionais: EMR e Redshift Spectrum

Esta etapa é apenas conceitual. **Não crie clusters, workgroups ou outros recursos pagos para executá-la.**

### Amazon EMR — Processamento distribuído

O Amazon EMR permite executar frameworks como Apache Spark para processar grandes volumes de dados.

Exemplo de operação que poderia ser realizada em um ambiente Spark:

```python
df = spark.sql("""
    SELECT *
    FROM db_lab7.vendas
    WHERE valor > 100
""")
```

O resultado poderia ser transformado e gravado novamente no S3 em formato Parquet.

### Amazon Redshift Spectrum — Consulta de dados externos

O Redshift Spectrum permite consultar dados armazenados no S3 utilizando tabelas externas integradas ao Glue Data Catalog.

Exemplo conceitual de consulta:

```sql
SELECT
    categoria,
    SUM(valor) AS faturamento_total
FROM redshift_lab7.vendas
GROUP BY categoria;
```

O esquema externo e o ambiente Redshift precisariam estar previamente configurados.

**Observação:** para este laboratório, o Athena já atende à necessidade de executar consultas SQL diretamente sobre os dados no S3. EMR e Redshift Spectrum representam alternativas para cenários com requisitos adicionais de processamento e data warehouse.

## 🧹 9. Limpeza dos recursos

Ao terminar o laboratório:

1. Acesse o bucket no **Amazon S3**.
2. Utilize a opção **Empty** para remover os arquivos de `raw/`, `curated/` e `athena-results/`.
3. Acesse **CloudFormation**.
4. Selecione a stack `datalake-lab7` e clique em **Delete**.
5. Caso tenha criado outros recursos durante os testes, remova-os também.
