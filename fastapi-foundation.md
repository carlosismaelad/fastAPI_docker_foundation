
# FastAPI - Configuração do Projeto

## 1. Criar a pasta do projeto

```bash
mkdir nome-do-projeto && cd nome-do-projeto
```

## 2. Criar o ambiente virtual

```bash
python -m venv .venv
source .venv/bin/activate
```

## 3. Criar o arquivo `requirements-dev.txt` com as ferramentas de desenvolvimento

```text
ipython             # Terminal interativo
ipdb                # Debugger
sdb                 # Debugger remoto
pip-tools           # Lock de dependências
pytest              # Execução de testes
pytest-order        # Ordenação de testes
httpx               # Cliente HTTP assíncrono para testes
black               # Autoformatação
flake8              # Linter
```

## 4. Atualizar o pip

```bash
pip install --upgrade pip
```

## 5. Instalar as dependências de desenvolvimento

```bash
pip install -r requirements-dev.txt
```

## 6. Inicializar o repositório Git

```bash
git init
```

## 7. Criar a estrutura de pastas

Organize de acordo com a arquitetura do seu projeto ou crie um arquivo `.py` com o conteúdo abaixo e execute-o:

```python
import os

estrutura = {
    ".": [
        "setup.py",
        "settings.toml",
        ".secrets.toml",
        "requirements.in",
        "MANIFEST.in",
        "Dockerfile.dev",
        "docker-compose.yaml"
    ],
    "postgres": [
        "Dockerfile",
        "create-databases.sh"
    ],
    "nome_do_projeto": [
        "default.toml",
        "__init__.py",
        "cli.py",
        "app.py",
        "auth.py",
        "db.py",
        "security.py",
        "config.py"
    ],
    "nome_do_projeto/models": [
        "__init__.py",
        "post.py",
        "user.py"
    ],
    "nome_do_projeto/routes": [
        "__init__.py",
        "auth.py",
        "post.py",
        "user.py"
    ],
    "tests": [
        "__init__.py",
        "conftest.py",
        "test_api.py"
    ]
}

for pasta, arquivos in estrutura.items():
    os.makedirs(pasta, exist_ok=True)
    for arquivo in arquivos:
        caminho = os.path.join(pasta, arquivo)
        open(caminho, 'a').close()

open("test.sh", 'a').close()

print("Estrutura do projeto criada com sucesso!")
```

## 8. Criar o arquivo `requirements.in`

Esse arquivo é usado apenas para declarar dependências, sem travar versões.

```text
fastapi                        # Framework web/API
uvicorn                        # Webserver ASGI para rodar aplicações FastAPI
sqlmodel                       # ORM baseado em SQLAlchemy e Pydantic
typer                          # CLI para administrar o projeto
dynaconf                       # Gerenciamento de configurações
jinja2                         # Template engine
python-jose[cryptography]     # JWT com criptografia
passlib[bcrypt]                # Hash de senhas
python-multipart               # Upload de arquivos
psycopg2-binary                # Driver PostgreSQL
alembic                        # Ferramenta de migração de banco de dados
rich                           # Terminal com visual mais agradável
```

> **Nota:** No `requirements.in` informamos o nome das bibliotecas sem versão. O `requirements.txt` terá as versões travadas após compilação.

## 9. Gerar `requirements.txt` a partir do `requirements.in`

```bash
pip-compile requirements.in
```

## 10. Criar o `app.py`

Configure título, versão e descrição da aplicação FastAPI.

```python
from fastapi import FastAPI

app = FastAPI(
    title="Titulo do app",
    version="0.1.0",
    description="Descrição do projeto",
)
```

## 11. Tornar o projeto instalável

### a. Criar o `MANIFEST.in`

Arquivo onde listamos os pacotes incluídos no pacote Python:

```text
graft nome-do-projeto
```

### b. Criar o `setup.py` na raiz do projeto

Inclui metadados e configuração para transformar o projeto em pacote Python.

```python
import io
import os
from setuptools import find_packages, setup

# função para entrada de arquivos 
def read(*paths, **kwargs):
    content = ''
    with io.open(
        os.path.join(os.path.dirname(__file__), *paths),
        encoding=kwargs.get("encoding", "utf8"),
    ) as open_file:
        content = open_file.read().strip()
    return content

# função para obter as dependências do projeto
def read_requirements(path):
    return [
        line.strip()
        for line in read(path).split("\n")
        if line and not line.startswith(('"', "#", "-", "git+"))
    ]

setup(
    name="nome-do-projeto",
    version="0.1.0",
    description="descrição do que é o projeto",
    url="https://nome_do_projeto.io",
    python_requires=">=3.12",
    long_description="Uma descrição mais detalhada do projeto",
    long_description_content_type="text/markdown",
    author="Marshall Mellow",
    packages=find_packages(exclude=["tests"]),
    include_package_data=True,
    install_requires=read_requirements("requirements.txt"),
    entry_points={
        "console_scripts": ["nome_do_projeto = nome_do_projeto.cli:main"]
    }
)
```

### c. Instalar o projeto localmente de forma editável

```bash
pip install -e .
```

## 12. Criar `Dockerfile.dev`

Você poderá usar o exemplo abaixo:

```dockerfile
# Builder app image
FROM python:3.12

#Create directory
RUN mkdir -p /home/app

# Create the app user
RUN groupadd app && useradd -g app app

#Create the home directory
ENV APP_HOME=/home/app/api
RUN mkdir -p $APP_HOME
WORKDIR $APP_HOME

# install
COPY . $APP_HOME
RUN pip install -r requirements-dev.txt
RUN pip install -e .

RUN chown -R app:app $APP_HOME
USER app

CMD [ "uvicorn", "nome_do_projeto.app:app", "--host=0.0.0.0", "--port=8000", "--reload" ]
```

Para gerar a imagem da aplicação:

```bash
docker build -f Dockerfile.dev -t nome-do-app:latest .
```

## 13. Testar aplicação com Docker

Adicione uma rota `/` no `app.py` e execute:

```bash
docker run --rm -it -v $(pwd):/home/app/api -p 8000:8000 nome-do-app
```

Explicações:
- `--rm`: remove o container ao parar
- `-it`: modo interativo
- `-v $(pwd):/home/app/api`: mapeia o diretório local para dentro do container

> Certifique-se de que o diretório `/home/app/api` exista no `Dockerfile.dev`.

Repositório de exemplo: [GitHub - carlosismaelad/app_fastAPI](https://github.com/carlosismaelad/app_fastAPI)

## 14. Configurar container do Postgres

Na pasta `./postgres`, crie:

### a. `create-databases.sh`

Exemplo de script para criação de bancos e usuários.

```bash
#!/bin/zsh

set -e
set -u

function create_user_and_database(){
    local database=$1
    echo "Creating user and database '$database'"
    psql -v ON_ERROR_STOP=1 --username "$POSTGRES_USER" <<-EOSQL
        CREATE USER $database PASSWORD '$database';
        CREATE DATABASE $database;
        GRANT ALL PRIVILEGES ON DATABASE $database TO $database;
EOSQL
}

if [-n "$POSTGRES_DBS"]; then
    echo "Creating DB(s): $POSTGRES_DBS"
    for db in $(echo $POSTGRES_DBS | tr ',' ' '); do
        create_user_and_database $db
    done
    echo "Multiple databases created"
fi
```

### b. `Dockerfile`

Exemplo de Dockerfile para subir o Postgres

```dockerfile
FROM postgres:alpine3.14
COPY create-databases.sh /docker-entrypoint-initdb.d/
```

```Dockerfile
FROM postgres:alpine3.14
COPY create-databases.sh /docker-entrypoint-initdb.d/
```

## 15. Criar `docker-compose.yaml`

Para orquestrar os containers da aplicação e do banco de dados.

Você pode nomeá-lo de docker-compose.yaml ou apenas compose.yaml.

Se desejar, use o exemplo abaixo:

```yaml
services:
  api:
    build:
      context: .
      dockerfile: Dockerfile.dev
    ports:
      - "8000:8000"
    environment:
      APP_DB_uri: "postgresql://postgres:postgres@db:5432/${APP_DB:-dbname}"
      APP_DB_connect_args: "{}"
    volumes:
      - .:/home/app/api
    depends_on:
      - db
    stdin_open: true
    tty: true
  db:
    build: postgres
    image: app_postgres-13-alpine-multi-user
    volumes:
      - $HOME/.postgres/app_db/data/postgresql:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    environment:
      - POSTGRES_DBS=dbname, dbname_test
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres
```

## 16. Subir containers

```bash
docker compose up
```

(use sem `-d` para visualizar os logs no terminal)

## 17. Definir a modelagem de dados inicial

Crie os modelos de dados usando `sqlmodel` e configure as tabelas.
