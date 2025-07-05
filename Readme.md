

\# API de Cadastro de Produtos com ASP.NET Core Identity e JWT



Este projeto implementa uma API RESTful para cadastro de produtos, incorporando autenticação e autorização via ASP.NET Core Identity e JSON Web Tokens (JWT).



\## 🚀 Como Configurar e Rodar o Projeto



Siga os passos abaixo para configurar o ambiente, aplicar as migrações e testar a API.



\### Pré-requisitos



Certifique-se de ter as seguintes ferramentas instaladas:



&nbsp; \* \[.NET SDK](https://dotnet.microsoft.com/download) (versão 8.0 ou superior)

&nbsp; \* \[SQL Server LocalDB](https://www.google.com/search?q=https://docs.microsoft.com/en-us/sql/relational-databases/sql-server-express-localdb%3Fview%3Dsql-server-ver16) (geralmente vem com o Visual Studio) ou outra instância do SQL Server.

&nbsp; \* \[Visual Studio](https://visualstudio.microsoft.com/) (recomendado) ou outro editor de código como \[VS Code](https://code.visualstudio.com/).

&nbsp; \* \[Postman](https://www.postman.com/downloads/) ou \[Swagger UI](https://www.google.com/search?q=http://localhost:5080/swagger/index.html) (já configurado no projeto) para testar os endpoints.



\### 📦 Instalação de Pacotes



Garanta que os seguintes pacotes NuGet estejam instalados no seu projeto `ApiProduto`:



&nbsp; \* \*\*Para ASP.NET Core Identity com Entity Framework Core:\*\*



&nbsp;   ```bash

&nbsp;   dotnet add package Microsoft.AspNetCore.Identity.EntityFrameworkCore --version 8.0.17

&nbsp;   ```



&nbsp;   \[]



&nbsp; \* \*\*Para autenticação JWT Bearer:\*\*



&nbsp;   ```bash

&nbsp;   dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer --version 8.0.17

&nbsp;   ```



&nbsp;   \[]



&nbsp; \* \*\*Para SQL Server com Entity Framework Core:\*\*



&nbsp;   ```bash

&nbsp;   dotnet add package Microsoft.EntityFrameworkCore.SqlServer

&nbsp;   ```



&nbsp; \* \*\*Para Ferramentas do Entity Framework Core (necessário para migrações):\*\*



&nbsp;   ```bash

&nbsp;   dotnet add package Microsoft.EntityFrameworkCore.Design

&nbsp;   ```



\### ⚙️ Configuração do Banco de Dados e Identity



1\.  \*\*Ajuste a String de Conexão:\*\*

&nbsp;   Abra o arquivo `appsettings.json` e configure sua string de conexão para o SQL Server LocalDB (ou sua instância de SQL Server):



&nbsp;   ```json

&nbsp;   {

&nbsp;     "ConnectionStrings": {

&nbsp;       "DefaultConnection": "Server=(LocalDb)\\\\MSSQLLocalDB;Database=ApiProduto;Trusted\_Connection=True;MultipleActiveResultSets=true"

&nbsp;     },

&nbsp;     "JwtSettings": {

&nbsp;       "Segredo": "SEU\_SEGREDO\_SUPER\_FORTE\_AQUI\_PELO\_MENOS\_16\_CARACTERES",

&nbsp;       "ExpiracaoHoras": 1,

&nbsp;       "Emissor": "MeuSistema",

&nbsp;       "Audiencia": "https://localhost"

&nbsp;     }

&nbsp;   }

&nbsp;   ```



&nbsp;   \*\*Importante:\*\* Adapte o `Server` da sua `DefaultConnection` se você não estiver usando a instância padrão do LocalDB.



2\.  \*\*Atualize o `ApiContext`:\*\*

&nbsp;   Altere a classe `ApiContext.cs` (localizada em `Data/ApiContext.cs`) para herdar de `IdentityDbContext`. Isso habilita o Entity Framework Core a gerenciar as tabelas de usuário, papéis e outras entidades do Identity.



&nbsp;   ```csharp

&nbsp;   using ApiProduto.Model; // Seu modelo Produto

&nbsp;   using Microsoft.EntityFrameworkCore;

&nbsp;   using Microsoft.AspNetCore.Identity.EntityFrameworkCore; // Adicione este using

&nbsp;   using Microsoft.AspNetCore.Identity; // Adicione este using



&nbsp;   namespace ApiProduto.Data

&nbsp;   {

&nbsp;       // Altere a herança para IdentityDbContext

&nbsp;       public class ApiContext : IdentityDbContext<IdentityUser, IdentityRole, string>

&nbsp;       {

&nbsp;           public ApiContext(DbContextOptions<ApiContext> options) : base(options)

&nbsp;           {

&nbsp;           }



&nbsp;           public DbSet<Produto> Produtos { get; set; }



&nbsp;           protected override void OnModelCreating(ModelBuilder modelBuilder)

&nbsp;           {

&nbsp;               // Chame o base.OnModelCreating PRIMEIRO para o Identity configurar suas tabelas

&nbsp;               base.OnModelCreating(modelBuilder);



&nbsp;               // Aplique suas configurações de entidade personalizadas (para Produto)

&nbsp;               modelBuilder.ApplyConfigurationsFromAssembly(typeof(ApiContext).Assembly);

&nbsp;           }

&nbsp;       }

&nbsp;   ```



3\.  \*\*Adicione a Configuração do Identity no `Program.cs`:\*\*

&nbsp;   Certifique-se de que seu arquivo `Program.cs` está configurando os serviços do Identity e do JWT corretamente. A implementação recomendada é usar um método de extensão como `AddIdentityConfig`.



&nbsp;   Exemplo do seu `Program.cs` (partes relevantes):



&nbsp;   ```csharp

&nbsp;   using ApiProduto.Configuration; // Onde AddIdentityConfig está

&nbsp;   using ApiProduto.Data;

&nbsp;   using Microsoft.EntityFrameworkCore;



&nbsp;   var builder = WebApplication.CreateBuilder(args);



&nbsp;   builder

&nbsp;       // ... outras configurações

&nbsp;       .AddIdentityConfig(builder.Configuration); // Garanta que IConfiguration seja passado



&nbsp;   // Registro do DbContext principal (pode ser repetido se o IdentityConfig também registrar)

&nbsp;   builder.Services.AddDbContext<ApiContext>(options =>

&nbsp;   {

&nbsp;       options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection"));

&nbsp;   });



&nbsp;   var app = builder.Build();



&nbsp;   // ... Middlewares de autenticação e autorização

&nbsp;   app.UseAuthentication();

&nbsp;   app.UseAuthorization();

&nbsp;   // ...

&nbsp;   ```



&nbsp;   E o método `AddIdentityConfig` (em `ApiProduto.Configuration/IdentityConfig.cs`):



&nbsp;   ```csharp

&nbsp;   public static class IdentityConfig

&nbsp;   {

&nbsp;       public static IServiceCollection AddIdentityConfig(this IServiceCollection services, IConfiguration configuration)

&nbsp;       {

&nbsp;           // Registra o DbContext para o Identity (se não for feito no Program.cs principal)

&nbsp;           services.AddDbContext<ApiContext>(options =>

&nbsp;               options.UseSqlServer(configuration.GetConnectionString("DefaultConnection")));



&nbsp;           // Adiciona o ASP.NET Core Identity

&nbsp;           services.AddDefaultIdentity<IdentityUser>()

&nbsp;               .AddRoles<IdentityRole>() // Para suporte a roles

&nbsp;               .AddEntityFrameworkStores<ApiContext>() // Usa o ApiContext para persistência

&nbsp;               .AddDefaultTokenProviders();



&nbsp;           // Configuração do JWT Bearer

&nbsp;           var jwtSettings = configuration.GetSection("JwtSettings").Get<JwtSettings>();

&nbsp;           // ... restante da configuração do JWT conforme instruído anteriormente

&nbsp;           // (JwtSettings e AddAuthentication().AddJwtBearer(...) )



&nbsp;           return services;

&nbsp;       }

&nbsp;   }

&nbsp;   ```



\### 💾 Migrações do Banco de Dados



Após as alterações no `ApiContext`, você precisa criar e aplicar uma nova migração para que as tabelas do Identity sejam geradas no seu banco de dados.



1\.  \*\*Abra o Package Manager Console\*\* (Visual Studio: `Tools` \\> `NuGet Package Manager` \\> `Package Manager Console`).

2\.  \*\*Defina o `Default project`\*\* para o projeto que contém seu `ApiContext.cs` (geralmente o seu projeto principal da API).

3\.  \*\*Remova migrações anteriores (apenas se for um projeto novo ou de desenvolvimento sem dados a preservar):\*\*

&nbsp;   ```powershell

&nbsp;   Remove-Migration

&nbsp;   ```

&nbsp;   (Se houver migrações, ele perguntará se deseja removê-las. Confirme.)

&nbsp;   \*\*CUIDADO:\*\* Se você já tem dados importantes, pode ser necessário criar uma migração incremental em vez de remover as antigas.

4\.  \*\*Adicione uma nova migração:\*\*

&nbsp;   ```powershell

&nbsp;   Add-Migration AddIdentityAndProductTables

&nbsp;   ```

&nbsp;   Este comando criará um arquivo de migração contendo o esquema para as tabelas do Identity e suas entidades de produto.

5\.  \*\*Aplique a migração ao banco de dados:\*\*

&nbsp;   ```powershell

&nbsp;   Update-Database

&nbsp;   ```

&nbsp;   Isso executará o script SQL gerado pela migração, criando as tabelas no seu banco de dados.



\### 🏃 Rodando a API



1\.  No Visual Studio, pressione `F5` ou clique em `Debug` \\> `Start Debugging`.

2\.  A API será iniciada e o Swagger UI será aberto no seu navegador (geralmente em `http://localhost:5080/swagger`).



\### 🔑 Testando a Autenticação JWT



1\.  \*\*Registro de Usuário:\*\*



&nbsp;     \* No Swagger UI, encontre o endpoint de `POST` para `api/auth/register` (ou o nome que você deu ao seu endpoint de registro).

&nbsp;     \* Envie um JSON com `Email` e `Password` para criar um novo usuário.

&nbsp;     \* Exemplo de corpo da requisição:

&nbsp;       ```json

&nbsp;       {

&nbsp;         "email": "teste@exemplo.com",

&nbsp;         "password": "Senha@123",

&nbsp;         "confirmPassword": "Senha@123"

&nbsp;       }

&nbsp;       ```

&nbsp;     \* Verifique a resposta para confirmar o sucesso.



2\.  \*\*Login e Obtenção do Token:\*\*



&nbsp;     \* Encontre o endpoint de `POST` para `api/auth/login`.

&nbsp;     \* Envie as credenciais do usuário recém-criado:

&nbsp;       ```json

&nbsp;       {

&nbsp;         "email": "teste@exemplo.com",

&nbsp;         "password": "Senha@123"

&nbsp;       }

&nbsp;       ```

&nbsp;     \* A resposta deve incluir o token JWT. Copie este token.



3\.  \*\*Validação do Token (Opcional):\*\*

&nbsp;   Você pode usar o site \[https://jwt.io/](https://jwt.io/) para colar o token e inspecionar seu conteúdo (header, payload e assinatura). Isso ajuda a entender os claims e a data de expiração.



4\.  \*\*Acessando Endpoints Protegidos:\*\*



&nbsp;     \* No Swagger UI, para endpoints que exigem autenticação (como `GET /api/produtos` se estiver protegido), clique no botão de autorização (geralmente um cadeado ou "Authorize").

&nbsp;     \* No campo de valor, insira `Bearer SEU\_TOKEN\_AQUI` (substitua `SEU\_TOKEN\_AQUI` pelo token que você copiou no passo de login).

&nbsp;     \* Agora, ao executar requisições para endpoints protegidos, elas devem ser autenticadas com sucesso.



-----



Este `README.md` deve cobrir os principais passos para configurar, rodar e testar sua API.

