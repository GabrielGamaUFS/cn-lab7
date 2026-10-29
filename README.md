
# 🚀 Laboratório Prático: Data Lake & Analytics na AWS — Lab 7

Este laboratório demonstra o fluxo de ingestão, limpeza, catalogação e análise de dados utilizando serviços da AWS.

## 🏗️ Arquitetura

```text
[CSV criado no AWS CloudShell]
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
- AWS CloudShell disponível e autorizado.
- Permissões para utilizar CloudFormation, S3, Glue, DataBrew e Athena.
- Role `LabRole` disponível para os serviços que precisarem de uma role de execução.
- Utilize a mesma região AWS durante todo o laboratório.

> **Atenção aos custos:** utilize apenas os recursos necessários, acompanhe as cobranças e exclua os recursos ao terminar. Não crie clusters EMR nem ambientes Redshift neste laboratório.

## 💻 2. Preparação dos arquivos no AWS CloudShell

Abra o **AWS CloudShell** pelo console da AWS. Os comandos desta etapa criarão os arquivos diretamente no ambiente Linux do CloudShell.

### Criar o template.yaml

Execute o comando completo:

```bash
cat > template.yaml << 'EOF'
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
EOF
```

### Criar o vendas_2026.csv

Execute:

```bash
cat > vendas_2026.csv << 'EOF'
id_venda,data_venda,cliente,valor,categoria
1001,2026-01-15,José Santana,250.00,Eletronicos
1002,2026-01-16,Maria Silva,,Eletronicos
1003,2026-01-16,Carlos Souza,120.50,Vestuario
1004,2026/01/17,Ana Costa,450.00,Casa
1005,2026-01-18,Paulo Santos,89.90,Vestuario
1006,,Lucas Lima,1200.00,Eletronicos
EOF
```

Confira se os arquivos foram criados:

```bash
ls -l template.yaml vendas_2026.csv
```

## ⚙️ 3. Provisionamento com AWS CloudFormation

O CloudFormation criará o bucket S3 e o banco de dados do Glue Catalog. Não será criada uma nova role IAM.

### Criar a stack

No CloudShell, execute:

```bash
aws cloudformation create-stack \
  --stack-name datalake-lab7 \
  --template-body file://template.yaml
```

Aguarde a conclusão:

```bash
aws cloudformation wait stack-create-complete \
  --stack-name datalake-lab7
```

Se o comando falhar, consulte **CloudFormation → Stacks → datalake-lab7 → Events** antes de prosseguir.

Consulte os outputs:

```bash
aws cloudformation describe-stacks \
  --stack-name datalake-lab7 \
  --query "Stacks[0].Outputs" \
  --output table
```

### Guardar o nome do bucket

Execute:

```bash
BUCKET=$(aws cloudformation describe-stacks \
  --stack-name datalake-lab7 \
  --query "Stacks[0].Outputs[?OutputKey=='BucketName'].OutputValue | [0]" \
  --output text)

echo "$BUCKET"
```

A variável `BUCKET` armazenará o nome do bucket para os próximos comandos na mesma sessão do CloudShell.

> O template não cria permissões IAM. Se o Learner Lab bloquear a operação por falta de permissões, interrompa o procedimento e verifique as restrições do ambiente.

## ☁️ 4. Ingestão no Amazon S3

Envie o CSV diretamente do CloudShell:

```bash
aws s3 cp vendas_2026.csv \
  "s3://$BUCKET/raw/vendas/vendas_2026.csv"
```

O AWS CLI criará os prefixos necessários durante o envio.

Para conferir:

```bash
aws s3 ls "s3://$BUCKET/raw/vendas/"
```

O arquivo estará disponível em:

```text
s3://<nome-do-bucket>/raw/vendas/vendas_2026.csv
```

Substitua `<nome-do-bucket>` pelo nome exibido pelo comando `echo "$BUCKET"` nos campos do console AWS.

## 🧹 5. Limpeza com AWS Glue DataBrew

### Criar o dataset

1. Abra o **AWS Glue DataBrew** na mesma região utilizada anteriormente.
2. Acesse **Datasets → Connect new dataset**.
3. Configure:
   - **Dataset name:** `ds-lab7-vendas-raw`
   - **Source:** Amazon S3
   - **S3 location:** `s3://<nome-do-bucket>/raw/vendas/vendas_2026.csv`
4. Substitua `<nome-do-bucket>` pelo nome real do bucket.
5. Salve o dataset e, se necessário, execute a perfilagem dos dados (*Run data profile*).

### Criar o projeto

1. Acesse **Projects → Create project**.
2. Configure:
   - **Project name:** `proj-lab7-limpeza-vendas`
   - **Dataset:** `ds-lab7-vendas-raw`
   - **Recipe:** crie uma nova receita, caso solicitado.
   - **Role:** `LabRole`
3. Crie o projeto e aguarde a abertura do editor visual.
4. Na receita, aplique as transformações:
   - Remova registros com `valor` ausente.
   - Remova registros com `data_venda` ausente.
   - Converta `data_venda` para data, tratando o formato `2026/01/17`.
   - Padronize as datas para `YYYY-MM-DD`.
5. Salve as transformações na receita.

> Os nomes das opções de transformação podem variar na interface. Confirme que o valor e a data estão sendo interpretados corretamente antes de executar o job.

### Criar e executar o job

1. Publique a receita, caso ainda não esteja publicada.
2. No projeto, acesse a opção de criação de job (*Create job*).
3. Configure:

| Campo | Valor |
| --- | --- |
| Job name | `job-lab7-clean-vendas` |
| Dataset | `ds-lab7-vendas-raw` |
| Output format | Parquet |
| Output location | `s3://<nome-do-bucket>/curated/vendas/` |
| IAM role | `LabRole` |

4. Substitua `<nome-do-bucket>` pelo nome real do bucket.
5. Salve e execute o job (*Create and run job* ou a opção equivalente).
6. Aguarde o status **Succeeded** antes de prosseguir.

## 🗂️ 6. Catalogação com AWS Glue Crawler

Antes de criar o crawler, confirme que o banco `db_lab7` existe em **AWS Glue → Data Catalog → Databases**.

1. Abra **AWS Glue → Crawlers**.
2. Clique em **Create crawler**.
3. Informe o nome `crawler-lab7-vendas-curated`.
4. Configure:
   - **Data source:** Amazon S3
   - **S3 path:** `s3://<nome-do-bucket>/curated/vendas/`
   - **IAM role:** `LabRole`, se disponível e autorizada.
   - **Target database:** `db_lab7`
5. Substitua `<nome-do-bucket>` pelo nome real do bucket.
6. Conclua a criação e execute o crawler.
7. Aguarde a conclusão e acesse **Data Catalog → Tables**.
8. Localize a tabela criada e anote seu nome efetivo.

Nos exemplos seguintes, será utilizado `vendas`. Caso o crawler tenha registrado outro nome, substitua-o nas consultas.

## 📊 7. Consultas com Amazon Athena

### Configurar o local de resultados

1. Abra o **Amazon Athena** na mesma região.
2. Acesse **Settings → Manage** no editor de consultas.
3. Em **Location of query result**, informe:

```text
s3://<nome-do-bucket>/athena-results/
```

4. Substitua `<nome-do-bucket>` pelo nome real do bucket.
5. Salve a configuração, se disponível. Se o workgroup impuser outro local de resultados, utilize a configuração permitida pelo ambiente.
6. Volte ao editor e selecione o banco de dados `db_lab7`.
7. Confirme o nome da tabela no painel de dados.

### Consulta por categoria

Cole a consulta no editor e clique em **Run**:

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

Execute uma consulta por vez. Se a tabela tiver outro nome, substitua `vendas` nas consultas.

Considerando as transformações propostas, o conjunto deverá conter quatro registros, com faturamento total de `910.40`.

## 🔬 8. Cenários opcionais: EMR e Redshift Spectrum

Esta etapa é apenas conceitual. **Não crie clusters EMR, workgroups adicionais ou ambientes Redshift para executá-la.**

### Amazon EMR — Processamento distribuído

O Amazon EMR permite executar frameworks como Apache Spark para processar grandes volumes de dados.

Exemplo de operação que poderia ser realizada em um ambiente Spark previamente configurado:

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

O esquema externo, as permissões e o ambiente Redshift precisariam estar previamente configurados.

**Observação:** para este laboratório, o Athena já atende à necessidade de executar consultas SQL diretamente sobre os dados no S3. EMR e Redshift Spectrum representam alternativas para cenários com requisitos adicionais de processamento e data warehouse.

## 🧹 9. Limpeza dos recursos

Ao terminar o laboratório, remova somente os recursos criados para o Lab 7.

### Limpeza manual

1. **DataBrew:** exclua o job `job-lab7-clean-vendas`, o projeto `proj-lab7-limpeza-vendas` e o dataset `ds-lab7-vendas-raw`. Confirme que o job não está em execução.
2. **Glue Crawler:** em **AWS Glue → Crawlers**, exclua `crawler-lab7-vendas-curated`.
3. **Glue Data Catalog:** se a tabela `vendas` tiver sido criada pelo crawler, exclua somente essa tabela, caso ainda exista.
4. **Amazon S3:** abra exclusivamente o bucket cujo nome foi exibido pelo output `BucketName` da stack `datalake-lab7`. Confira o nome antes de prosseguir e utilize **Empty** para remover os arquivos do laboratório.
5. **CloudFormation:** acesse **Stacks**, selecione `datalake-lab7` e clique em **Delete**. Aguarde a conclusão da exclusão.
