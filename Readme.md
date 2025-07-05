
# API de Cadastro de Produtos com ASP.NET Core Identity e JWT

Este projeto implementa uma API RESTful para cadastro de produtos, incorporando autenticação e autorização via ASP.NET Core Identity e JSON Web Tokens (JWT).

## 🚀 Como Configurar e Rodar o Projeto

Siga os passos abaixo para configurar o ambiente, aplicar as migrações e testar a API.

### Pré-requisitos

Certifique-se de ter as seguintes ferramentas instaladas:

  * [.NET SDK](https://dotnet.microsoft.com/download) (versão 8.0 ou superior)
  * [SQL Server LocalDB](https://www.google.com/search?q=https://docs.microsoft.com/en-us/sql/relational-databases/sql-server-express-localdb%3Fview%3Dsql-server-ver16) (geralmente vem com o Visual Studio) ou outra instância do SQL Server.
  * [Visual Studio](https://visualstudio.microsoft.com/) (recomendado) ou outro editor de código como [VS Code](https://code.visualstudio.com/).
  * [Postman](https://www.postman.com/downloads/) ou [Swagger UI](https://www.google.com/search?q=http://localhost:5080/swagger/index.html) (já configurado no projeto) para testar os endpoints.

### 📦 Instalação de Pacotes

Garanta que os seguintes pacotes NuGet estejam instalados no seu projeto `ApiProduto`:

  * **Para ASP.NET Core Identity com Entity Framework Core:**

    ```bash
    dotnet add package Microsoft.AspNetCore.Identity.EntityFrameworkCore --version 8.0.17
    ```

 

  * **Para autenticação JWT Bearer:**

    ```bash
    dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer --version 8.0.17
    ```
 

  * **Para SQL Server com Entity Framework Core:**

    ```bash
    dotnet add package Microsoft.EntityFrameworkCore.SqlServer
    ```

  * **Para Ferramentas do Entity Framework Core (necessário para migrações):**

    ```bash
    dotnet add package Microsoft.EntityFrameworkCore.Design
    ```

### ⚙️ Configuração do Banco de Dados e Identity

1.  **Ajuste a String de Conexão:**
    Abra o arquivo `appsettings.json` e configure sua string de conexão para o SQL Server LocalDB (ou sua instância de SQL Server):

    ```json
    {
      "ConnectionStrings": {
        "DefaultConnection": "Server=(LocalDb)\\MSSQLLocalDB;Database=ApiProduto;Trusted_Connection=True;MultipleActiveResultSets=true"
      },
      "JwtSettings": {
        "Segredo": "SEU_SEGREDO_SUPER_FORTE_AQUI_PELO_MENOS_16_CARACTERES",
        "ExpiracaoHoras": 1,
        "Emissor": "MeuSistema",
        "Audiencia": "https://localhost"
      }
    }
    ```

    **Importante:** Adapte o `Server` da sua `DefaultConnection` se você não estiver usando a instância padrão do LocalDB.

2.  **Atualize o `ApiContext`:**
    Altere a classe `ApiContext.cs` (localizada em `Data/ApiContext.cs`) para herdar de `IdentityDbContext`. Isso habilita o Entity Framework Core a gerenciar as tabelas de usuário, papéis e outras entidades do Identity.

    ```csharp
    using ApiProduto.Model; // Seu modelo Produto
    using Microsoft.EntityFrameworkCore;
    using Microsoft.AspNetCore.Identity.EntityFrameworkCore; // Adicione este using
    using Microsoft.AspNetCore.Identity; // Adicione este using

    namespace ApiProduto.Data
    {
        // Altere a herança para IdentityDbContext
        public class ApiContext : IdentityDbContext<IdentityUser, IdentityRole, string>
        {
            public ApiContext(DbContextOptions<ApiContext> options) : base(options)
            {
            }

            public DbSet<Produto> Produtos { get; set; }

            protected override void OnModelCreating(ModelBuilder modelBuilder)
            {
                // Chame o base.OnModelCreating PRIMEIRO para o Identity configurar suas tabelas
                base.OnModelCreating(modelBuilder);

                // Aplique suas configurações de entidade personalizadas (para Produto)
                modelBuilder.ApplyConfigurationsFromAssembly(typeof(ApiContext).Assembly);
            }
        }
    ```

3.  **Adicione a Configuração do Identity no `Program.cs`:**
    Certifique-se de que seu arquivo `Program.cs` está configurando os serviços do Identity e do JWT corretamente. A implementação recomendada é usar um método de extensão como `AddIdentityConfig`.

    Exemplo do seu `Program.cs` (partes relevantes):

    ```csharp
    using ApiProduto.Configuration; // Onde AddIdentityConfig está
    using ApiProduto.Data;
    using Microsoft.EntityFrameworkCore;

    var builder = WebApplication.CreateBuilder(args);

    builder
        // ... outras configurações
        .AddIdentityConfig(builder.Configuration); // Garanta que IConfiguration seja passado

    // Registro do DbContext principal (pode ser repetido se o IdentityConfig também registrar)
    builder.Services.AddDbContext<ApiContext>(options =>
    {
        options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection"));
    });

    var app = builder.Build();

    // ... Middlewares de autenticação e autorização
    app.UseAuthentication();
    app.UseAuthorization();
    // ...
    ```

    E o método `AddIdentityConfig` (em `ApiProduto.Configuration/IdentityConfig.cs`):

    ```csharp
    public static class IdentityConfig
    {
        public static IServiceCollection AddIdentityConfig(this IServiceCollection services, IConfiguration configuration)
        {
            // Registra o DbContext para o Identity (se não for feito no Program.cs principal)
            services.AddDbContext<ApiContext>(options =>
                options.UseSqlServer(configuration.GetConnectionString("DefaultConnection")));

            // Adiciona o ASP.NET Core Identity
            services.AddDefaultIdentity<IdentityUser>()
                .AddRoles<IdentityRole>() // Para suporte a roles
                .AddEntityFrameworkStores<ApiContext>() // Usa o ApiContext para persistência
                .AddDefaultTokenProviders();

            // Configuração do JWT Bearer
            var jwtSettings = configuration.GetSection("JwtSettings").Get<JwtSettings>();
            // ... restante da configuração do JWT conforme instruído anteriormente
            // (JwtSettings e AddAuthentication().AddJwtBearer(...) )

            return services;
        }
    }
    ```

### 💾 Migrações do Banco de Dados

Após as alterações no `ApiContext`, você precisa criar e aplicar uma nova migração para que as tabelas do Identity sejam geradas no seu banco de dados.

1.  **Abra o Package Manager Console** (Visual Studio: `Tools` \> `NuGet Package Manager` \> `Package Manager Console`).
2.  **Defina o `Default project`** para o projeto que contém seu `ApiContext.cs` (geralmente o seu projeto principal da API).
3.  **Remova migrações anteriores (apenas se for um projeto novo ou de desenvolvimento sem dados a preservar):**
    ```powershell
    Remove-Migration
    ```
    (Se houver migrações, ele perguntará se deseja removê-las. Confirme.)
    **CUIDADO:** Se você já tem dados importantes, pode ser necessário criar uma migração incremental em vez de remover as antigas.
4.  **Adicione uma nova migração:**
    ```powershell
    Add-Migration AddIdentityAndProductTables
    ```
    Este comando criará um arquivo de migração contendo o esquema para as tabelas do Identity e suas entidades de produto.
5.  **Aplique a migração ao banco de dados:**
    ```powershell
    Update-Database
    ```
    Isso executará o script SQL gerado pela migração, criando as tabelas no seu banco de dados.

### 🏃 Rodando a API

1.  No Visual Studio, pressione `F5` ou clique em `Debug` \> `Start Debugging`.
2.  A API será iniciada e o Swagger UI será aberto no seu navegador (geralmente em `http://localhost:5080/swagger`).

### 🔑 Testando a Autenticação JWT

1.  **Registro de Usuário:**

      * No Swagger UI, encontre o endpoint de `POST` para `api/auth/register` (ou o nome que você deu ao seu endpoint de registro).
      * Envie um JSON com `Email` e `Password` para criar um novo usuário.
      * Exemplo de corpo da requisição:
        ```json
        {
          "email": "teste@exemplo.com",
          "password": "Senha@123",
          "confirmPassword": "Senha@123"
        }
        ```
      * Verifique a resposta para confirmar o sucesso.

2.  **Login e Obtenção do Token:**

      * Encontre o endpoint de `POST` para `api/auth/login`.
      * Envie as credenciais do usuário recém-criado:
        ```json
        {
          "email": "teste@exemplo.com",
          "password": "Senha@123"
        }
        ```
      * A resposta deve incluir o token JWT. Copie este token.

3.  **Validação do Token (Opcional):**
    Você pode usar o site [https://jwt.io/](https://jwt.io/) para colar o token e inspecionar seu conteúdo (header, payload e assinatura). Isso ajuda a entender os claims e a data de expiração.

4.  **Acessando Endpoints Protegidos:**

      * No Swagger UI, para endpoints que exigem autenticação (como `GET /api/produtos` se estiver protegido), clique no botão de autorização (geralmente um cadeado ou "Authorize").
      * No campo de valor, insira `Bearer SEU_TOKEN_AQUI` (substitua `SEU_TOKEN_AQUI` pelo token que você copiou no passo de login).
      * Agora, ao executar requisições para endpoints protegidos, elas devem ser autenticadas com sucesso.

 
