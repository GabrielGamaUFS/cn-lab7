# 🚀 Laboratório Prático: Data Lake \& Analytics na AWS — Lab 7

Este laboratório demonstra o fluxo de ingestão, limpeza, catalogação e análise de dados utilizando serviços da AWS.

## 🏗️ Arquitetura

``` text
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

> **Atenção aos custos:** utilize apenas os recursos necessários, acompanhe as cobranças e exclua os recursos ao terminar. Não crie clusters EMR nem ambientes Redshift neste laboratório.

## 💻 2. Preparação dos arquivos no AWS CloudShell

Abra o **AWS CloudShell** pelo console da AWS. Os comandos desta etapa criarão os arquivos diretamente no ambiente Linux do CloudShell.

### Criar o template.yaml

Execute o comando completo:

``` bash
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

### Criar o vendas\_2026.csv

Execute:

``` bash
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

Os arquivos estão prontos. Se quiser conferir, execute:

``` bash
ls -l template.yaml vendas_2026.csv
```

## ⚙️ 3. Provisionamento com AWS CloudFormation

O CloudFormation criará o bucket S3 e o banco de dados do Glue Catalog. Não será criada uma nova role IAM.

### Criar a stack

No CloudShell, execute:

``` bash
aws cloudformation create-stack \
  --stack-name datalake-lab7 \
  --template-body file://template.yaml
```

Aguarde a conclusão:

``` bash
aws cloudformation wait stack-create-complete \
  --stack-name datalake-lab7
```

Consulte os outputs:

``` bash
aws cloudformation describe-stacks \
  --stack-name datalake-lab7 \
  --query "Stacks[0].Outputs" \
  --output table
```

### Guardar o nome do bucket

Execute:

``` bash
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

``` bash
aws s3 cp vendas_2026.csv \
  "s3://$BUCKET/raw/vendas/vendas_2026.csv"
```

O AWS CLI criará os prefixos necessários durante o envio.

Para conferir:

``` bash
aws s3 ls "s3://$BUCKET/raw/vendas/"
```

O arquivo estará disponível em:

``` text
s3://<nome-do-bucket>/raw/vendas/vendas_2026.csv
```

## 🧹 5. Limpeza com AWS Glue DataBrew

### Criar o dataset

1. Abra o **AWS Glue DataBrew**.
2. Acesse **Datasets → Connect new dataset**.
3. Configure:
   - **Dataset name:** `ds-lab7-vendas-raw`
   - **Source:** Amazon S3
   - **S3 location:** `s3://<nome-do-bucket>/raw/vendas/vendas_2026.csv`
4. Salve o dataset e execute a perfilagem dos dados (*Run data profile*).

### Criar o projeto

1. Acesse **Projects → Create project**.
2. Configure:
   - **Project name:** `proj-lab7-limpeza-vendas`
   - **Dataset:** `ds_lab7_vendas_raw`
   - **Role:** `LabRole`
3. Na interface visual, aplique as transformações:
   - Remova registros com `valor` ausente.
   - Remova registros com `data_venda` ausente.
   - Padronize as datas para `YYYY-MM-DD`, tratando o formato `2026/01/17`.

### Criar o job

Crie um job a partir da receita:

| Campo | Valor |
| --- | --- |
| Job name | `job-lab7-clean-vendas` |
| Output format | Parquet |
| Output location | `s3://<nome-do-bucket>/curated/vendas/` |

Publique a receita, se solicitado, e execute o job.

Aguarde a conclusão do job antes de prosseguir.

## 🗂️ 6. Catalogação com AWS Glue Crawler

1. Abra **AWS Glue → Crawlers**.
2. Crie um crawler chamado `crawler-lab7-vendas-curated`.
3. Configure:
   - **Data source:** Amazon S3
   - **S3 path:** `s3://<nome-do-bucket>/curated/vendas/`
   - **IAM role:** `LabRole`, se disponível e autorizada.
   - **Target database:** `db_lab7`
4. Conclua a criação e execute o crawler.
5. Após a execução, acesse **Data Catalog → Tables** e localize a tabela criada.

Anote o nome efetivo da tabela. Nos exemplos seguintes, será utilizado `vendas`; caso o crawler tenha registrado outro nome, substitua-o nas consultas.

## 📊 7. Consultas com Amazon Athena

### Configurar o local de resultados

1. Abra o **Amazon Athena**.
2. Acesse as configurações do editor (*Settings*).
3. Configure o local de saída:

``` text
s3://<nome-do-bucket>/athena-results/
```

4. Selecione o banco de dados `db_lab7`.

### Consulta por categoria

``` sql
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

``` sql
SELECT COUNT(*) AS total_registros
FROM "db_lab7"."vendas";
```

**Faturamento total:**

``` sql
SELECT ROUND(SUM(valor), 2) AS faturamento_total
FROM "db_lab7"."vendas";
```

Considerando as transformações propostas, o conjunto deverá conter quatro registros, com faturamento total de `910.40`.

## 🔬 8. Cenários opcionais: EMR e Redshift Spectrum

Esta etapa é apenas conceitual. **Não crie clusters, workgroups ou outros recursos pagos para executá-la.**

### Amazon EMR — Processamento distribuído

O Amazon EMR permite executar frameworks como Apache Spark para processar grandes volumes de dados.

Exemplo de operação que poderia ser realizada em um ambiente Spark:

``` python
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

``` sql
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

### Limpeza de todos serviços utilizados (⚠️Atenção: Irá excluir todos os CloudFormations, Glue, DataBrew criados na sua AWS)

``` bash
echo "=== 1. Apagando todos os recursos do AWS DataBrew ==="
for job in $(aws databrew list-jobs --query "Jobs[].Name" --output text 2>/dev/null); do
    aws databrew delete-job --name "$job" 2>/dev/null
done
for proj in $(aws databrew list-projects --query "Projects[].Name" --output text 2>/dev/null); do
    aws databrew delete-project --name "$proj" 2>/dev/null
done
for ds in $(aws databrew list-datasets --query "Datasets[].Name" --output text 2>/dev/null); do
    aws databrew delete-dataset --name "$ds" 2>/dev/null
done

echo "=== 2. Apagando todos os Crawlers do AWS Glue ==="
for crawler in $(aws glue get-crawlers --query "Crawlers[].Name" --output text 2>/dev/null); do
    aws glue delete-crawler --name "$crawler" 2>/dev/null
done

echo "=== 3. Esvaziando e deletando Buckets S3 do Lab ==="
BUCKETS=$(aws s3api list-buckets --query "Buckets[?contains(Name, 'datalake') || contains(Name, 'lab')].Name" --output text)
for b in $BUCKETS; do
    if [ "$b" != "None" ] && [ -n "$b" ]; then
        echo "Esvaziando s3://$b..."
        aws s3 rm "s3://$b" --recursive
        aws s3api delete-bucket --bucket "$b" 2>/dev/null || true
    fi
done

echo "=== 4. Deletando Databases residuais no Glue Catalog ==="
for db in $(aws glue get-databases --query "DatabaseList[].Name" --output text 2>/dev/null); do
    if [[ "$db" == *"lab"* || "$db" == *"datalake"* ]]; then
        aws glue delete-database --name "$db" 2>/dev/null || true
    fi
done

echo "=== 5. Deletando Stacks do CloudFormation ==="
for stack in $(aws cloudformation list-stacks --stack-status-filter CREATE_COMPLETE UPDATE_COMPLETE --query "StackSummaries[].StackName" --output text 2>/dev/null); do
    if [[ "$stack" == *"datalake"* || "$stack" == *"lab"* ]]; then
        echo "Deletando stack $stack..."
        aws cloudformation delete-stack --stack-name "$stack"
    fi
done

echo "=== LIMPEZA COMPLETA FINALIZADA COM SUCESSO! ==="
```
``` bash
rm -rf template.yaml vendas_2026.csv
```
### Limpeza Manual:

1. No Amazon S3, abra o bucket e utilize **Empty** para remover os arquivos de `raw/`, `curated/` e `athena-results/`.
2. No CloudFormation, selecione a stack `datalake-lab7` e clique em **Delete**.
3. Remova os Datasets e Projects do DataBrew, e o Crawlers do Glue.
