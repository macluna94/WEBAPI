# Notas de ASP.NET

Comandos para crear un nuevo proyecto en .NET Core

#### Creacion del entorno del proyecto

**Creacion de un nuevo proyecto**

Crear una carpeta donde se ubicara el proyecto. El comando de creacion tomara el nombre de la capreta de forma predeterminada

`C:/> dotnet new sln`

**Crear clases complementarias para el proyecto**

Son adisiones que complementan el funcionamiento del la API

- Dominio [ _Modelos_ ]
- Persistencia
- Aplicacion
- WAPI [ _Interfaz_ ]

`C:/> dotnet new classlib -o Modelo`

`C:/> dotnet new classlib -o Persistencia`

`C:/> dotnet new classlib -o Aplicacion`

`C:/> dotnet new webapi -o WAPI`

**Agregar las  clases complemento al proyecto**
Hay que agregar al proyecto la relacion de la clases creadas

`C:/> dotnet sln add Dominio/`
`C:/> dotnet sln add Persistencia/`
`C:/> dotnet sln add Aplicacion/`
`C:/> dotnet sln add WAPI/`

**Inyeccion de Dependencias en Aplicacion**

Se agrega la dependencia de Dominio a **Persistencia**

`C:/Persistencia/> dotnet add  reference ../Dominio`

Se agrega la dependencia de Dominio y Persistencia en **Aplicacion**

`C:/Aplicacion/> dotnet add  reference ../Dominio`

`C:/Aplicacion/> dotnet add  reference ../Persistencia`

Se agrega la dependencia de Aplicacion a **WAPI**

`C:/WAPI> dotnet add  reference ../Aplicacion`

#### Configuracion de acceso a la base de datos

**Conexion a la base de datos**

En el archivo *appsettings.json* se agrega el siguiente codigo:

```
"ConnectionStrings":{
    "DefaultConnection": "Data Source=localhost\\SQLEXPRESS;Initial Catalog=BibliotecaDB;Integrated Security=True "
  }
```

En _Catalog=BibliotecaDB se coloca el nombre de la base de datos


**Creamos el contexto para manejar el acceso a los datos**

En *Persistencia* creamos una clase para crear un contexto

Con **NUGET** se instala:

- Microsoft.EntityFrameworkCore ->[3.15]
- Microsoft.EntityFrameworkCore.SqlServer ->[3.15]

```
	public void ConfigureServices(IServiceCollection services)
		{
    		services.AddDbContext<BiblioDbContext>(options => {
        		options.UseSqlServer(Configuration.GetConnectionString("DefaultConnection"));
	         });
    	  services.AddControllers();
	}
```


#### Creacion del Modelo

Se usan una clase para crear un modelo para cada tabla con los mismos campos y del mismo tipo

En *Persistencia/Context* se setea el modelo *Dbset\<Usuario\> Usuario {get; set;}*


#### Creacion de la logica de Negocio

En Aplicacion se crean las funciones o procesos que manipularan la base de datos y realizaran los procesos de la API

#### Validacion de campos 

**Por Anotaciones**
Los campos pueden validarse mediante **Anotaciones** [Required (ErrorMessage="Falta Titulo")]

```cs
    public class Ejecuta: IRequest {
    	[Required (ErrorMessage="Falta Titulo")]
        public string Titulo {
            get;
            set;
        }
        public String Descripcion {
            get;
            set;
        }
        public DateTime FechaCreacion {
            get;
            set;
        }
    }

```

**FluentValidation**

Con la libreria *FluentValidation* se haran las validaciones, se instala el modulo en Aplicacion:

- FluentValidation.AspNetCore

Y se configura un metodo dentro de _Startup.cs_ 

`services.AddControllers().AddFluentValidation(cfg => cfg.RegisterValidatorsFromAssemblyContaining<Nuevo>());`

En el metodo a validar se crea una clase que ejecutara la logica de validacion

```cs
    public class EjecutaValidation: AbstractValidator < Ejecuta > {
                public EjecutaValidation() {
                    RuleFor(x => x.Titulo).NotEmpty();
                    RuleFor(x => x.Descripcion).NotEmpty();
                    RuleFor(x => x.FechaCreacion).NotEmpty();
                }
            }

```

Aqui se crean las reglas de validacion: _RuleFor(x => x.Titulo).**NotEmpty()**;_

**Descripciones de errores middleware**

Se puede complenetar la validacion de otros errores de la aplicacion. Mediante un middleware podemos complentar la gestion de errores

Clase manejadora en Aplicacion
`Aplicacion/ManejadorError/ManException.cs`

**ManException.cs**

En esta clase se almacenaran los mensajes

```cs
    namespace Aplicacion.ManejadorError
	{
		public class ManException : Exception
		{
			public HttpStatusCode Codigo { get; }
			public object Errores { get; }

        	public ManException(HttpStatusCode codigo, object errores = null)
			{
				this.Codigo = codigo;
				this.Errores = errores;
			}
		}
	}
```

**MAnErrorMiddle.cs**

`WebAPI/Middleware/ManErrorMiddle.cs`

En esta clase de creara un middleware que interceptara las Responses y capturara los errores o dara aso a las transacciones:

```cs

    public class ManErrorMiddle
    {
        private readonly RequestDelegate _next;
        private readonly ILogger<ManErrorMiddle> _logger;
        // Clase que recoge el contexto y los mensajes de la transaccion
        public ManErrorMiddle(RequestDelegate next, ILogger<ManErrorMiddle> logger)
        {
            _next = next;
            _logger = logger;
        }

        /// Interceptor del Request 
        public async Task Invoke(HttpContext context)
        {
            try
            {

                // Si todo va bien pasa
                await _next(context);
            }
            catch (Exception ex)
            {
                // Si hay Error se ejecuta
                await ManejadorExceptionAsync(context, ex, _logger);
            }
        }

        // Recolector y despachador de errores al usuario
        public async Task ManejadorExceptionAsync(HttpContext context, Exception ex, ILogger<ManErrorMiddle> logger)
        {
            object errores = null;

            switch (ex)
            {
                // Si es error de http
                case ManException man:
                    logger.LogError(ex, "Manejador error");
                    errores = man.Errores;
                    context.Response.StatusCode = (int)man.Codigo;
                    break;
                // Si es un error de otra indole
                case Exception e:
                    logger.LogError(ex, "Manejador de Servidor");
                    errores = string.IsNullOrWhiteSpace(e.Message) ? "Error" : e.Message;
                    context.Response.StatusCode = (int)HttpStatusCode.InternalServerError;
                    break;
            }
            // La Respuesta sera en Json
            context.Response.ContentType = "application/json";
            if (errores != null)
            {
                var result = JsonConvert.SerializeObject(new { errores });
                await context.Response.WriteAsync(result);
            }
        }
    }
```

En _Startup.cs_ se configura el middleware


```cs

 public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
 {
 	// Aplicacion del middleware propio
	app.UseMiddleware<ManErrorMiddle>();

```

Implenetacion de una excepcion personalizada

```cs
 if (curso == null)
                {
                    // aplicacion de una excepcion propia
                    throw new ManException(HttpStatusCode.NotFound, new { Mensaje = "no se encontro curso" });

                }

```


#### Actualizacion de liberias

En los **.csproj** las versiones tiene que coincidir con la webapi es por eso que se tienen que actualizar


```xml

  <PropertyGroup>
    <TargetFramework>netstandard2.0</TargetFramework>
  </PropertyGroup>
```

Se sustituye `netstandard2.0` por `netcoreapp3.1` y se agrega unas nuevas etiquetas

```xml
  <PropertyGroup>
    <TargetFramework>netcoreapp3.1</TargetFramework>
  </PropertyGroup>
  <ItemGroup>
    <FrameworkReference Include="Microsoft.AspNetCore.App" />
  </ItemGroup>

```

Ya modificado se ejecutan los siguientes comandos:
`dotnet restore`
`dotnet build`



#### Migracion de Tablas

Las clases-modelo puede crear tablas

Hay que preparar los datos para que sean interpretados y convertidos a codigo SQL

*Ids autogenerados*

```cs
	public int CursoId{ get; set;}
	public int InstructorId{ get; set;}
    
    //---------------------
	// los id usan un tipo de datos autogenerado de identificacion "Guid" 
    public Guid CursoId{ get; set;}
    public Guid InstructorId{ get; set;}
```

*Especificacion de longitud*

Se coloca una anotacion donde se especifica la longitud del campo

```cs
	[Column(TypeName="decimal(18,4)")]
	public decimal PrecioActual{ get; set; }
```

Configuracion en **Program.cs**
```cs
 public static void Main(string[] args)
        {
            var hostserver = CreateHostBuilder(args).Build();

            //Creacion de un contexto o ambiente
            using (var ambiente = hostserver.Services.CreateScope())
            {
                var services = ambiente.ServiceProvider;

                try
                {
                    var context = services.GetRequiredService<CursosContext>();
                    // Migracion
                    context.Database.Migrate();
                }
                catch (Exception ex)
                {
                    // En caso de un error
                    var logging = services.GetRequiredService<ILogger<Program>>();
                    logging.LogError(ex, "ERROR EN LA MIGRACION");
                }

                // Ya que finalice la Migracion o no
                hostserver.Run();
            }

        }

```





**Migraciones**

**Herramienta** para la creacion de tablas o migracion

`dotnet tool install --global dotnet-ef --version 3.1.2`


**Generacion de los archivos de Migracions**
`dotnet ef migrations add IdentityCoreInicial -p Persistencia/ -s WebAPI/ia/ -s WebAPI/`

**Ejecucion de la Migracion**

`\WebAPI>dotnet watch run`


_ _ _


#### Seguridad Identity Core

Para agregar la seguridad con Identity es necesario instarlar **_Microsoft.AspNetCore.Identity.EntityFrameworkCore@3.1.2_** y agregarlo a la clase _Persistencia/CursoDbContext.cs_ . En esta heredara de la clase **IdentityDbContext**


```cs
	public class CursosContext: IdentityDbContext<Usuario>
    {
    }
```

Configuracion de servicios para la creacion del Identity y la administracion de login 

```cs
    public void ConfigureServices(IServiceCollection services)
        {
            // Configuracion de acceso a la base de datos
            services.AddDbContext<CursosContext>(opt => {
                opt.UseSqlServer(Configuration.GetConnectionString("DefaultConnection"));
            });
            // Agregar Mediador
            services.AddMediatR(typeof(Consulta.Manejador).Assembly);
            // Agregando la libreria FluentValidation
            services.AddControllers().AddFluentValidation(cfg => cfg.RegisterValidatorsFromAssemblyContaining<Nuevo>());

            //Configuracion del Identity<Usuario>
            var builder = services.AddIdentityCore<Usuario>();
            var indentity_Builder = new IdentityBuilder(builder.UserType, builder.Services);
            indentity_Builder.AddEntityFrameworkStores<CursosContext>();
            // Set del login
            indentity_Builder.AddSignInManager<SignInManager<Usuario>>();

        }
```

Ususario de prueba con un metodo temporal para la inserccion de datos

```cs
    namespace Persistencia
    {
        public class DataPrueba
        {
              public static async Task InsertarDataAsync(CursosContext context, UserManager<Usuario> usuarioManager)
        {
            if (!usuarioManager.Users.Any())
            {
                var usuario = new Usuario
                {
                    NombreCompleto = " Mac Luna",
                    UserName = "macluna94",
                    Email = "macluna94@gmail.com"
                };

                await usuarioManager.CreateAsync(usuario,"p123456789" );
            }
        }
           }
```

Configurando **Program.cs**

```cs
  	var userManager = services.GetRequiredService<UserManager<Usuario>>();
    var context = services.GetRequiredService<CursosContext>();
    // Migracion
    context.Database.Migrate();

    //Establecimiento de la inserccion
    await DataPrueba.InsertarDataAsync(context, userManager);
```

Se agregan un metodo para la autenticacion en **Startup.cs**

```cs
    //Configuracion del Identity<Usuario>
    var builder = services.AddIdentityCore<Usuario>();
    var indentity_Builder = new IdentityBuilder(builder.UserType, builder.Services);
    indentity_Builder.AddEntityFrameworkStores<CursosContext>();
    // Set del login
    indentity_Builder.AddSignInManager<SignInManager<Usuario>>();

	//---------------------------------------------------------------------//

    // Metodos para el funcionamiento de autenticacion y encriptacion
    services.TryAddSingleton<ISystemClock, SystemClock>();
```











_ _ _























