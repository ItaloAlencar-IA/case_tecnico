# Case Técnico

## 1. Documentação do Processo de Importação

1.2 Ferramentas utilizadas para importação
GitHub (disponibilizar estrutura simplificada do projeto)
VS Code (criar vínculos com Docker, GitHub e Apache Hop)
Docker (vinculação e execução de ferramentas em contêineres isolados)
Apache Hop (ETL - Pipeline e Workflow)
Dbeaver (fazer as consultas e queries)

Criado relação, mas não utilizado devido tempo - Jenkins (gestão de CI e CD)

### 1.3 Estrutura das Planilhas (colunas e tipos de dados)

```
venda(
    venda_id int8 NOT NULL,
    data_emissao date NOT NULL,
    horariomov varchar(8) DEFAULT '00:00:00'::character varying NOT NULL,
    produto_id varchar(25) DEFAULT ''::character varying NOT NULL,
    qtde_vendida float8 NULL,
    valor_unitario numeric(12, 4) DEFAULT 0 NOT NULL,
    filial_id int8 DEFAULT 1 NOT NULL,
    item int4 DEFAULT 0 NOT NULL,
    unidade_medida varchar(3) NULL,
    CONSTRAINT pk_consumo PRIMARY KEY (filial_id, venda_id, data_emissao, produto_id, item, horariomov)
``` 
```
pedido_compra(
    pedido_id float8 DEFAULT 0 NOT NULL,
    data_pedido date NULL,
    item float8 DEFAULT 0 NOT NULL,
    produto_id varchar(25) DEFAULT '0' NOT NULL,
    descricao_produto varchar(255) NULL,
    ordem_compra float8 DEFAULT 0 NOT NULL,
    qtde_pedida float8 NULL,
    filial_id int4 NULL,
    data_entrega date NULL,
    qtde_entregue float8 DEFAULT 0 NOT NULL,
    qtde_pendente float8 DEFAULT 0 NOT NULL,--não informada na planilha
    preco_compra float8 DEFAULT 0 NULL,
    fornecedor_id int4 DEFAULT 0 NULL,
    CONSTRAINT pedido_compra_pkey PRIMARY KEY (pedido_id , produto_id, item)
```
```
entradas_mercadoria (
	ordem_compra float8 DEFAULT 0 NOT NULL, 
	data_entrada date NULL,
	nro_nfe varchar(255) NOT NULL,
	item float8 DEFAULT 0 NOT NULL,
	produto_id varchar(25) DEFAULT '0' NOT NULL,
	descricao_produto varchar(255) NULL,
	qtde_recebida float8 NULL,
	filial_id int4 NULL,
	custo_unitario numeric(12, 4) DEFAULT 0 NOT NULL,
	CONSTRAINT entradas_mercadoria_pkey PRIMARY KEY (ordem_compra, item, produto_id, nro_nfe)
```
```
produtos_filial (
	filial_id int4 NOT NULL,
	produto_id varchar(25) DEFAULT '0' NOT NULL, --não informada na planilha com esse nome
	descricao varchar(255) NOT NULL,
	estoque float8 DEFAULT 0 NOT NULL, -- caso haja produto não inteiro, numeric(18,3) NOT NULL DEFAULT 0
	preco_unitario float8 DEFAULT 0 NOT NULL, -- alterar para (numeric(12,4) NOT NULL DEFAULT 0)
	preco_compra float8 DEFAULT 0 NOT NULL, -- alterar para (numeric(12,4) NOT NULL DEFAULT 0)
	preco_venda float8 DEFAULT 0 NOT NULL, -- alterar para (numeric(12,4) NOT NULL DEFAULT 0)
	fornecedor_id int4 null, --não informada na planilha com esse nome e alterar tipo de dado (varchar(25) NOT NULL)
	CONSTRAINT produtos_filial_pkey PRIMARY KEY (filial_id, produto_id)
```
```
fornecedor(
    idfornecedor varchar(25) NOT NULL,
    razao_social varchar(255) NOT NULL,
    CONSTRAINT fornecedor_pkey PRIMARY KEY (idfornecedor, razao_social)
```

### 1.4 Tratamentos aplicados (conversão de tipos, validações, normalizações)
Todas as colunas de data já formatadas para "DD/MM/YYYY" (via apache hop)
Trigger de fornecedor inserido (via apache hop)


### 1.5 Ajustes ou correções realizadas durante o processo
Alterado nome das colunas via pipeline
tabela produtos filial (produto_id, fornecedor_id)
colunas que não existiam foram criadas (quantidade total)


## 2. Consultas sql básicas

### 2.1. Total de vendas em quantidade e em valores por cada produto no mês de fevereiro de 2025

```
WITH input AS (
    SELECT
    2025::int AS ano, 2::int AS mes -- altere aqui o ano e mês para consulta
)

select
  v.produto_id,
  v.qtde_vendida,
  SUM(v.qtde_vendida * v.valor_unitario) AS total_vendas
FROM venda v
    cross join input i
WHERE 
    extract(year from v.data_emissao) = i.ano
    AND extract(month from v.data_emissao) = i.mes
    GROUP BY v.produto_id, v.qtde_vendida
    ORDER BY total_vendas desc, v.qtde_vendida desc;
```
### 2.2 Pedidos requisitados mas não existe ordem de compra

```
select * from public.pedido_compra pc
where
pc.ordem_compra = 0; -- Requisitados mas não existe ordem de compra
```

### 3. Transformações de dados

### 3.1. Concatenar os campos produto_id e descricao_produto (onde houver) no formato;

```
select 
    produto_id || ' - ' || descricao_produto as ID_Produto,
    *
from pedido_compra pc
```

```
select 
    v.produto_id || ' - ' || pc.descricao_produto as ID_Produto,
    v.*
from public.venda v
    left JOIN pedido_compra pc
    on pc.produto_id = v.produto_id
```

### 3.2. Transformar o campo de datas para o formato `DD/MM/YYYY`; 

Feito para todos via Apache Hop

### 3.3. Retornar os dados filtrando apenas os produtos requisitados mais de 10 vezes no período.

```
select 
v.produto_id || ' - ' || pc.descricao_produto as ID_Produto,
v.*
from public.venda v
left JOIN pedido_compra pc
on pc.produto_id = v.produto_id
where
v.qtde_vendida > 10 
```

### 3.4. Criar uma trigger que gere automaticamente um novo `idfornecedor` numérico na tabela de produtos que se relacione com a tabela de fornecedor.

Feito via Apache Hop no pipeline da tabela de fornecedor


### 4. Estratégia de validação com o cliente

### 4.1. Quais seriam os principais pontos que você validaria com o cliente?
Faria perguntas relacionadas a 3 relações diretas: 
 - O que é cada coluna... o que seria a quantidade, porque alguns são quebrados, se impacta diretamente na unidade_medida e o que seria a data_emissão
 - Se referente ao que eles possuem de informação o resultado é semelhante ou idêntico para comparativos
 - Existe algum caso de histórico de mudança em algum dos produtos sinalizados
 - Caso houvesse possibilidade iria fazer a correlação da tabela de compras referente a alguns itens estarem vindo de forma incorreta
 
### 4.2. Quais técnicas utilizaria para garantir a exatidão e a precisão dos dados?
Um dos métodos foi o que foi sinalizado acima resultado feito x cliente, relação entre tabelas, ETL bem consolidado (tratamento de datas, colunas que causem estranheza).

### 4.3. Quais consultas você deixaria prontas para usar na reunião de validação?
Total de vendas em quantidade e em valores por cada produto no mês de fevereiro de 2025, também deixaria ela somente sem total somente com o histórico de fevereiro e exploraria caso necessário, comparação tabela produtos _filial x venda (sinalizando que existem produtos em venda que não estão no produtos_filial)