# Agente Text-to-SQL

Projeto desenvolvido para a atividade de GenAI do Rocket Lab 2026.2.

O objetivo é permitir que usuários realizem consultas analíticas sobre o catálogo de filmes da CineData Analytics utilizando perguntas em linguagem natural, sem a necessidade de conhecer SQL.

O agente recebe uma pergunta, identifica as informações relevantes do banco de dados, gera uma consulta SQL de leitura, executa a consulta sobre a camada Gold e retorna a resposta em linguagem natural.

## Tecnologias utilizadas

- Python
- Jupyter Notebook
- SQLite
- PydanticAI
- Pydantic
- OpenRouter
- NVIDIA Nemotron 3.5 Lightning
- python-dotenv

## Estrutura do projeto

```text
gen-AI/
├── app/
│   ├── cinerocket.db
│   └── text_to_sql.ipynb
├── .env
├── .env.example
├── .gitignore
├── requirements.txt
└── README.md
```

> O arquivo `cinerocket.db` e o arquivo `.env` não são versionados no repositório.

## Arquitetura do agente

O projeto utiliza um pipeline Text-to-SQL inspirado em etapas de sistemas de geração e validação de consultas.

O fluxo principal é:

```text
Pergunta em linguagem natural
        ↓
Schema Linking
        ↓
Construção do contexto focado
        ↓
Geração do SQL
        ↓
Validação de segurança
        ↓
Execução read-only no SQLite
        ↓
Verificação do resultado
        ↓
Correção, quando necessária
        ↓
Resposta final
```

### Schema Linking

O agente identifica quais tabelas, colunas, filtros e relacionamentos são necessários para responder à pergunta.

Essa etapa reduz a quantidade de informações enviadas ao modelo nas etapas seguintes.

### Contexto focado

A partir do resultado do Schema Linking, são recuperadas apenas as informações relevantes do banco, incluindo:

- colunas necessárias;
- colunas utilizadas em joins;
- amostras de valores;
- valores categóricos reais;
- relacionamentos entre tabelas.

Essa etapa ajuda a reduzir alucinações durante a geração do SQL.

### Geração do SQL

O modelo gera uma consulta compatível com SQLite utilizando somente as tabelas e colunas fornecidas pelo contexto.

O agente é instruído a gerar apenas consultas de leitura.

### Validação e execução

Antes da execução, o SQL passa por uma validação que bloqueia comandos de modificação, como:

- `INSERT`
- `UPDATE`
- `DELETE`
- `DROP`
- `ALTER`
- `CREATE`

Além disso, a conexão com o SQLite é realizada em modo somente leitura.

### Verificação e correção

Após a execução, o resultado pode ser analisado por um agente de verificação semântica.

Caso a consulta seja considerada inadequada, o pipeline realiza uma tentativa de correção utilizando:

- a pergunta original;
- o SQL gerado;
- o problema identificado;
- o contexto relevante do banco.

Para evitar loops e consumo excessivo de requisições, é realizada apenas uma tentativa de correção.

### Fallback

Como modelos gratuitos disponibilizados pelo OpenRouter podem apresentar indisponibilidade temporária, o projeto utiliza novas tentativas controladas nas chamadas ao modelo.

Na etapa de verificação e na geração da resposta final também existem mecanismos de fallback para evitar que uma falha temporária do provider interrompa completamente uma consulta que já foi executada com sucesso.


## Como executar o projeto

### 1. Clonar o repositório

```bash
git clone URL_DO_REPOSITORIO
cd gen-AI
```

### 2. Criar um ambiente virtual

No macOS/Linux:

```bash
python3 -m venv .venv
source .venv/bin/activate
```

No Windows:

```bash
python -m venv .venv
.venv\Scripts\activate
```

### 3. Instalar as dependências

```bash
pip install -r requirements.txt
```

### 4. Configurar a chave da OpenRouter

Crie um arquivo `.env` na raiz do projeto.

Você pode utilizar o `.env.example` como referência:

```env
OPENROUTER_API_KEY=sua_chave_api
```

Substitua `sua_chave_api` pela sua chave da OpenRouter.


### 5. Adicionar o banco de dados

Baixe o arquivo `cinerocket.db` disponibilizado na atividade e coloque-o dentro da pasta:

```text
app/
```

A estrutura deve ficar:

```text
app/
├── cinerocket.db
└── text_to_sql.ipynb
```

*O banco não está incluído no repositório por estar configurado no `.gitignore`.*

### 6. Executar o notebook

Abra:

```text
app/text_to_sql.ipynb
```

em um ambiente compatível com Jupyter Notebook, como o VS Code.

Selecione o ambiente virtual criado anteriormente como kernel e execute as células em ordem.

*OBS: Algumas seções do notebook possuem células utilizadas apenas para TESTES intermediários, inspeção de resultados e validação das etapas do pipeline. Essas células ajudam a demonstrar o funcionamento de cada componente, mas não fazem parte obrigatória da execução principal do agente.*

## Modelo utilizado

O projeto utiliza o modelo:

```text
nvidia/nemotron-3.5-lightning:free
```

através da API da OpenRouter.

A configuração é feita pelo PydanticAI utilizando uma interface compatível com a API da OpenAI.


## Exemplo de execução

Pergunta:

```text
Qual dupla ator-diretor mais trabalhou junta?
```

O pipeline gera e executa uma consulta SQL e produz uma resposta em linguagem natural baseada exclusivamente no resultado retornado pelo banco.

Além da resposta final, o notebook permite visualizar:

- SQL gerado;
- registros retornados;
- resultado da verificação;
- indicação de correção da consulta.

## Bateria de testes

O notebook inclui uma bateria de testes com perguntas de diferentes níveis de complexidade.

São avaliados casos envolvendo:

- ordenação e limite;
- agregações;
- filtros;
- múltiplos joins;
- tabelas bridge;
- filtros temporais.

Cada teste apresenta a resposta final, SQL gerado, registros retornados, resultado da verificação e informação sobre eventual correção.

## Segurança

O agente foi desenvolvido exclusivamente para consultas de leitura.

Entre as medidas utilizadas estão:

- validação dos comandos SQL;
- bloqueio de comandos de escrita;
- conexão SQLite em modo read-only;
- uso somente de tabelas e colunas existentes;
- recuperação de valores categóricos diretamente do banco;
- resposta final baseada nos dados efetivamente retornados pela consulta.

## Limitações

O projeto utiliza um modelo gratuito através da OpenRouter.

Endpoints gratuitos podem apresentar indisponibilidade ou respostas inválidas temporariamente. Por esse motivo, o projeto utiliza retries controlados e mecanismos de fallback.

Também é importante considerar que uma única pergunta pode utilizar múltiplas chamadas ao modelo devido às diferentes etapas do pipeline.