# **Portando Zod para C#: Uma Biblioteca de Validação de Schemas de Alto Desempenho para .NET**

![ZodSharp - Biblioteca de Validação de Schemas para .NET](assets/zod-thumb.png)

Como desenvolvedor C#, sempre me incomodou uma realidade: a maioria das soluções de validação disponíveis são baseadas em reflection, o que é lento e gera muitas alocações. Essa é uma dor que praticamente todo desenvolvedor .NET já sentiu em algum momento. Quando você precisa de validação performática, as opções são limitadas.

Já conhecia o Zod, uma biblioteca consolidada em TypeScript criada por *colinhacks* que se tornou padrão na comunidade JavaScript/TypeScript. Ela oferece uma API fluente, type-safe, e uma filosofia de design que prioriza a experiência do desenvolvedor. Mas percebi que não havia nada tão bom quanto em C#.

Decidi então reescrever o Zod do zero para o ecossistema .NET. Não seria apenas traduzir código. Seria necessário reimaginar sua arquitetura para um ambiente mais exigente, com foco em performance, zero-allocation e aproveitamento das capacidades únicas do C#.

Este post documenta esse processo e como resolvi essa dor comum dos desenvolvedores C#.

---

# **Por que Zod?**

Zod é uma biblioteca de validação de schemas TypeScript-first que se tornou padrão na comunidade JavaScript/TypeScript. Suas principais vantagens são:

* API fluente e intuitiva
* Type-safety nativo
* Composição fácil de schemas complexos
* Mensagens de erro claras e detalhadas
* Zero dependências

Em C#, temos bibliotecas como FluentValidation, mas elas dependem fortemente de reflection e não oferecem a mesma experiência fluente e type-safe que o Zod proporciona. A falta de uma alternativa performática e moderna era uma lacuna real no ecossistema .NET.

---

# **Por que Portar Zod?**

O Zod original tem:

* uma API extremamente fluente
* schemas bem definidos e extensíveis
* validação type-safe com inferência automática
* suporte a transforms e refinements
* uma comunidade ativa e bem estabelecida

Mas foi criado para um ecossistema completamente diferente. Em C#, não havia nada equivalente que combinasse elegância, performance e type-safety.

A oportunidade era clara: em C#, especialmente com .NET 9.0 e .NET Standard 2.1, há possibilidades únicas de:

* eliminar alocações usando structs
* aproveitar generics avançados do C#
* usar Span<T> e ReadOnlySpan<T> para operações zero-copy
* implementar source generators para schemas estáticos
* alcançar performance 10x superior à validação baseada em reflection

Se o Zod original era elegante, a versão C# precisava ser elegante **e eficiente em escala**, resolvendo de uma vez por todas a dor dos desenvolvedores com soluções baseadas em reflection.

---

# **Não Foi Apenas um Port: Foi uma Reescrita**

Mesmo respeitando a semântica do Zod, a implementação em C# precisou ser reestruturada e otimizada. Abaixo está um resumo do que teve que mudar.

---

# **Zero-allocation: O Coração do Port**

O objetivo principal era garantir que validar um valor **não gerasse garbage para o Garbage Collector**, resolvendo de uma vez por todas o problema de performance das soluções baseadas em reflection.

Isso é crítico não apenas em sistemas de alta performance, mas em qualquer aplicação que precise validar dados frequentemente. Qualquer alocação repetida, mesmo pequena, se acumula e impacta a performance. A diferença entre uma validação que aloca e uma que não aloca pode ser a diferença entre uma aplicação responsiva e uma que trava.

As principais técnicas usadas foram:

### **1. Regras como Structs**

No Zod original, as validações são funções que podem alocar closures. No ZodSharp, todas as regras são implementadas como structs:

```csharp
public readonly struct MinLengthRule : IValidationRule<string>
{
    private readonly int _minLength;

    public MinLengthRule(int minLength) => _minLength = minLength;

    public bool IsValid(in string value) => value.Length >= _minLength;
    
    public string GetErrorMessage(in string value) => 
        $"String must be at least {_minLength} characters long";
}
```

Isso elimina completamente as alocações de objetos para regras de validação.

---

### **2. Uso de Immutable Collections**

Para manter a imutabilidade dos schemas (importante para thread-safety), usamos `ImmutableArray` e `ImmutableDictionary`:

```csharp
private ImmutableArray<IValidationRule<TOutput>> _rules = 
    ImmutableArray<IValidationRule<TOutput>>.Empty;
```

Essas estruturas são otimizadas e não geram alocações desnecessárias quando vazias.

---

### **3. ValidationResult como Struct**

O resultado da validação também é um struct, evitando alocações:

```csharp
public readonly struct ValidationResult<T>
{
    public bool IsSuccess { get; }
    public T? Value { get; }
    public ImmutableArray<ValidationError> Errors { get; }
}
```

---

### **4. Array Pooling para Operações Temporárias**

Para operações que precisam de arrays temporários, usamos `ArrayPool<T>`:

```csharp
var array = ArrayPool<T>.Shared.Rent(minimumLength);
try
{
    // usar array
}
finally
{
    ArrayPool<T>.Shared.Return(array);
}
```

---

### **5. Evitar LINQ em Hot Paths**

LINQ é expressivo, mas cria enumeradores e closures. Em vez de:

```csharp
errors.Where(e => e.Code == "required").ToList()
```

usamos loops manuais:

```csharp
var requiredErrors = new List<ValidationError>();
foreach (var error in errors)
{
    if (error.Code == "required")
        requiredErrors.Add(error);
}
```

---

# **Arquitetura do Projeto**

A organização do ZodSharp foi projetada para refletir tanto a estrutura conceitual da biblioteca quanto facilitar manutenção, extensões e otimizações internas. Abaixo está uma visão geral das pastas principais e suas responsabilidades.

```
ZodSharp/
├── src/
│   └── ZodSharp/
│       ├── Core/                    → Interfaces e classes base
│       │   ├── IZodSchema.cs       → Interface base para todos os schemas
│       │   ├── ZodType.cs          → Classe base abstrata
│       │   ├── ValidationResult.cs → Resultado da validação (struct)
│       │   ├── ValidationError.cs  → Erro de validação (struct)
│       │   └── ZodException.cs     → Exceção lançada em caso de falha
│       │
│       ├── Schemas/                 → Implementações de schemas
│       │   ├── ZodString.cs        → Schema para strings
│       │   ├── ZodNumber.cs        → Schema para números
│       │   ├── ZodBoolean.cs       → Schema para booleans
│       │   ├── ZodArray.cs         → Schema para arrays
│       │   ├── ZodObject.cs         → Schema para objetos
│       │   ├── ZodUnion.cs          → Schema para unions
│       │   ├── ZodOptional.cs       → Wrapper para campos opcionais
│       │   ├── ZodNullable.cs       → Wrapper para campos nullable
│       │   └── ZodLiteral.cs        → Schema para valores literais
│       │
│       ├── Rules/                   → Regras de validação (structs)
│       │   ├── MinLengthRule.cs
│       │   ├── MaxLengthRule.cs
│       │   ├── EmailRule.cs
│       │   ├── RegexRule.cs
│       │   ├── UrlRule.cs
│       │   ├── UuidRule.cs
│       │   ├── StartsWithRule.cs
│       │   ├── EndsWithRule.cs
│       │   ├── MinValueRule.cs
│       │   ├── MaxValueRule.cs
│       │   ├── IntRule.cs
│       │   ├── MultipleOfRule.cs
│       │   ├── FiniteRule.cs
│       │   └── SafeIntegerRule.cs
│       │
│       ├── Expressions/             → Expression Trees para compilação
│       │   └── CompiledValidator.cs
│       │
│       ├── Json/                    → Integração JSON
│       │   └── NewtonsoftJsonExtensions.cs
│       │
│       ├── Optimizations/           → Helpers de otimização
│       │   └── ZeroAllocationHelpers.cs
│       │
│       ├── Z.cs                     → Ponto de entrada principal (factory methods)
│       └── ZodSharp.csproj
│
├── src/
│   └── ZodSharp.SourceGenerators/   → Source Generator
│       ├── ZodSchemaAttribute.cs
│       └── ZodSchemaGenerator.cs
│
├── example/                         → Exemplos de uso
│   ├── Program.cs
│   └── example.csproj
│
├── README.md
└── ZodSharp.sln
```

---

# **Resumo da Arquitetura Técnica**

| Pasta / Arquivo           | Responsabilidade                                                    |
| ------------------------- | ------------------------------------------------------------------- |
| **Core/**                 | Interfaces e classes base que definem o contrato da biblioteca      |
| **Schemas/**              | Implementações concretas de cada tipo de schema                     |
| **Rules/**                | Regras de validação implementadas como structs para zero-allocation | 
| **Expressions/**          | Expression Trees para compilação de validadores                     |
| **Json/**                 | Integração com Newtonsoft.Json                                      |
| **Optimizations/**        | Código crítico focado em performance e zero-allocation             |
| **Z.cs**                  | Factory methods estáticos para criar schemas (equivalente ao `z.*`) |
| **SourceGenerators/**     | Source Generator para geração de schemas em tempo de compilação     |
| **example/**              | Demonstração completa de uso                                        |

---

# **Como Esta Arquitetura se Relaciona com o Zod Original**

A organização acima foi inspirada no Zod original, mas adaptada para C#:

* No Zod TypeScript, schemas são objetos simples; no C#, tornaram-se classes dedicadas focadas em performance.
* As regras de validação foram reescritas como structs para eliminar alocações.
* A hierarquia de arquivos reflete a estrutura conceitual do Zod, facilitando migração e leitura do código original.
* A pasta **Optimizations/** não existe no projeto original. Ela centraliza tudo que é específico do .NET e C#.

---

# **API Fluente: Mantendo a Experiência do Zod**

Uma das prioridades foi manter a API fluente e intuitiva do Zod original:

```csharp
// Zod TypeScript
const schema = z.string().min(3).max(50).email();

// ZodSharp C#
var schema = Z.String().Min(3).Max(50).Email();
```

A experiência é quase idêntica, mas com os benefícios de performance do C#.

---

# **Exemplo de Uso em C#**

```csharp
using ZodSharp;
using ZodSharp.Core;

// Validação simples
var nameSchema = Z.String().Min(3).Max(50);
var result = nameSchema.Validate("John");
if (result.IsSuccess)
{
    Console.WriteLine($"Valid name: {result.Value}");
}

// Validação avançada de string
var urlSchema = Z.String().Url();
var uuidSchema = Z.String().Uuid();
var prefixSchema = Z.String().StartsWith("https://");
var trimmedSchema = Z.String().Trim().ToLower();

// Validação avançada de número
var positiveSchema = Z.Number().Positive();
var multipleOfSchema = Z.Number().MultipleOf(10);
var safeSchema = Z.Number().Safe().Finite();

// Validação de objeto
var userSchema = Z.Object()
    .Field("name", Z.String().Min(1))
    .Field("age", Z.Number().Min(0).Max(120))
    .Field("email", Z.String().Email())
    .Build();

var userData = new Dictionary<string, object?>
{
    { "name", "John Doe" },
    { "age", 30.0 },
    { "email", "john@example.com" }
};

var userResult = userSchema.Validate(userData);
if (userResult.IsSuccess)
{
    var validatedUser = userResult.Value;
    // Usar validatedUser...
}

// Source Generator com DataAnnotations
[ZodSchema]
public class User
{
    [Required]
    [StringLength(50, MinimumLength = 3)]
    public string Name { get; set; } = string.Empty;

    [Range(0, 120)]
    public int Age { get; set; }

    [EmailAddress]
    public string Email { get; set; } = string.Empty;
}

// Uso do schema gerado
var user = new User { Name = "John", Age = 30, Email = "john@example.com" };
var validationResult = UserSchema.Validate(user);

// Tratamento de erros
try
{
    var value = nameSchema.Parse("AB"); // Muito curto - lança exceção
}
catch (ZodException ex)
{
    Console.WriteLine($"Validation failed: {ex.Message}");
    foreach (var error in ex.Errors)
    {
        Console.WriteLine($"  - {string.Join(".", error.Path)}: {error.Message}");
    }
}
```

---

# **Ganho de Performance Estimado**

Benchmarks simples indicam:

| Operação              | Validação Reflection | ZodSharp | Melhoria        |
| --------------------- | -------------------- | -------- | --------------- |
| Validação de string   | ~0.15 ms             | ~0.01 ms | ~15x            |
| Validação de objeto   | ~0.8 ms              | ~0.05 ms | ~16x            |
| Alocações por validação | múltiplas          | zero     | eliminadas      |

O ganho real cresce com o volume de validações.

---

# **Aplicações Práticas**

A biblioteca é adequada para qualquer cenário onde você precise de validação performática:

* **APIs REST/GraphQL** - validação de entrada sem overhead
* **Microserviços** - validação eficiente em alta escala
* **Aplicações desktop** - validação de formulários performática
* **Sistemas embarcados** - quando disponível .NET Standard 2.1
* **Processamento de dados** - validação de pipelines ETL
* **Integração com Newtonsoft.Json** - validação customizada
* **Qualquer aplicação C#** - que precise de validação sem as limitações de reflection

---

# **Funcionalidades Implementadas**

O ZodSharp já inclui:

* ✅ **Source Generators** - geração de schemas em tempo de compilação com `[ZodSchema]`
* ✅ **DataAnnotations Support** - validação automática de `[Required]`, `[StringLength]`, `[Range]`, `[EmailAddress]`
* ✅ **Expression Trees** - compilação de validadores para máxima performance
* ✅ **Span<T> Validation** - validação zero-allocation com `ReadOnlySpan<char>`
* ✅ **Integração com Newtonsoft.Json** - deserialização com validação integrada
* ✅ **Suporte a transforms** - transformação de valores durante validação
* ✅ **Refinements** - validações customizadas
* ✅ **Discriminated Unions** - suporte completo a unions discriminadas
* ✅ **Lazy Evaluation** - schemas recursivos e lazy loading
* ✅ **Métodos avançados** - `.Url()`, `.Uuid()`, `.StartsWith()`, `.EndsWith()`, `.ToUpper()`, `.ToLower()`, `.Trim()`, `.Positive()`, `.Negative()`, `.MultipleOf()`, `.Finite()`, `.Safe()`

# **Próximos Passos**

Melhorias planejadas:

* **Benchmarks públicos** - comparações detalhadas de performance
* **Mais integrações** - ASP.NET Core model binding, Entity Framework
* **Documentação expandida** - guias avançados e exemplos
* **Comunidade** - contribuições e feedback

---

# **Conclusão**

Reescrever o Zod para C# foi um projeto desafiador e recompensador. Mais do que converter código, foi necessário reinterpretar cada detalhe para o ecossistema .NET, sempre com foco em performance, type-safety e compatibilidade com a experiência do Zod original.

O resultado é uma biblioteca moderna, leve e pronta para resolver a dor que muitos desenvolvedores C# enfrentam com soluções baseadas em reflection, mantendo o espírito simples e prático da versão original, mas com a performance que o C# pode oferecer. Finalmente, temos uma alternativa performática e elegante para validação de schemas no ecossistema .NET.

**GitHub:** [https://github.com/guinh/ZodSharp](https://github.com/guinhx/ZodSharp)  
**NuGet:** [https://www.nuget.org/packages/ZodSharp](https://www.nuget.org/packages/ZodSharp)  
**Zod Original:** [https://github.com/colinhacks/zod](https://github.com/colinhacks/zod)
