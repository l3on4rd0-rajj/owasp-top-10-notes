# Gerencie suas Dependências no GitHub

Projetos de software normalmente dependem de pacotes ou dependências externas. Gerenciar essas dependências consome recursos, afeta a produtividade e, quando feito de forma inadequada, pode introduzir riscos de segurança na cadeia de suprimentos de software.

## Por que isso importa

Uma vulnerabilidade é uma falha no código de um projeto que pode ser explorada. Essas explorações podem comprometer:

- a **confidencialidade**;
- a **integridade**;
- a **disponibilidade** do projeto ou de outros projetos que utilizam esse código.

Nem sempre essas vulnerabilidades são percebidas rapidamente, porque muitas delas existem fora do código em que a equipe trabalha diretamente.

**Entender como gerenciar dependências com eficiência no GitHub melhora a segurança da cadeia de suprimentos de software.**

## O gráfico de dependências

O gráfico de dependências é um resumo dos arquivos de manifesto e de bloqueio armazenados em um repositório, bem como de quaisquer dependências submetidas ao repositório por meio da API de submissão de dependências.

Esses arquivos contêm metadados sobre o projeto.

O gráfico de dependências é gerado automaticamente para repositórios públicos e inclui:

- **Dependências**: ecossistema e pacotes dos quais o repositório depende.
- **Dependentes**: repositórios e pacotes que dependem do repositório.

## Tipos de dependência mostrados

O gráfico utiliza as informações dos arquivos de bloqueio e manifesto para listar dois tipos de dependências:

- **Dependências diretas**: definidas explicitamente em um arquivo de manifesto ou de bloqueio, ou enviadas por meio da API de envio de dependências.
- **Dependências indiretas**: também chamadas de dependências transitivas ou subdependências, utilizadas por pacotes que, por sua vez, são dependências do projeto.

---

## Como ativar o gráfico de dependências em repositórios privados

Como administrador do repositório, é possível ativar o gráfico de dependências em repositórios privados com estes passos:

1. Acesse o repositório no GitHub.
2. Selecione as configurações do repositório.
3. No menu à esquerda, em **Segurança**, selecione **Segurança Avançada**.
4. Na seção **Gráfico de dependências**, selecione **Ativar**.

## Como visualizar o gráfico de dependências

Para visualizar o gráfico:

1. Acesse o repositório no GitHub.
2. Selecione **Insights** do repositório.
3. Selecione o **gráfico de dependências**.
4. Escolha a aba **Dependências** ou **Dependentes**.

---

## Ecossistemas de pacotes suportados

Os formatos abaixo são recomendados para tornar o gráfico de dependências mais preciso. Em geral, o uso de **lock files** é preferível porque define as versões exatas das dependências diretas e indiretas usadas no projeto.

Se o repositório usar apenas arquivo de manifesto, as dependências indiretas podem não ser incluídas nas verificações de vulnerabilidades.

| Gerenciador de pacotes | Linguagens | Formatos recomendados | Todos os formatos suportados |
| --- | --- | --- | --- |
| Cargo | Rust | `Cargo.lock` | `Cargo.toml`, `Cargo.lock` |
| Composer | PHP | `composer.lock` | `composer.json`, `composer.lock` |
| NuGet | .NET (C#, F#, VB), C++ | `.csproj`, `.vbproj`, `.nuspec`, `.vcxproj`, `.fsproj` | `.csproj`, `.vbproj`, `.nuspec`, `.vcxproj`, `.fsproj`, `packages.config` |
| GitHub Actions | YAML | `.yml`, `.yaml` | `.yml`, `.yaml` |
| Go Modules | Go | `go.mod` | `go.mod` |
| Maven | Java, Scala | `pom.xml` | `pom.xml` |
| npm | JavaScript | `package-lock.json` | `package-lock.json`, `package.json` |
| pip | Python | `requirements.txt`, `pipfile.lock` | `requirements.txt`, `pipfile`, `pipfile.lock`, `setup.py` |
| pnpm | JavaScript | `pnpm-lock.yaml` | `pnpm-lock.yaml`, `package.json` |
| pub | Dart | `pubspec.lock` | `pubspec.yaml`, `pubspec.lock` |
| Poetry | Python | `poetry.lock` | `poetry.lock`, `pyproject.toml` |
| RubyGems | Ruby | `Gemfile.lock` | `Gemfile.lock`, `Gemfile`, `*.gemspec` |
| Swift Package Manager | Swift | `Package.resolved` | `Package.resolved` |

## Ponto-chave

Sempre que possível, mantenha os **arquivos de bloqueio versionados no repositório** e alinhados entre os colaboradores. Isso melhora a visibilidade, a consistência do ambiente e a capacidade do GitHub de detectar vulnerabilidades.
