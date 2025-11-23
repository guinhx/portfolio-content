# **Portando o Mistreevous para C#: Uma biblioteca de árvores de comportamento de alto desempenho para .NET moderno.**

Quando comecei a trabalhar em servidores dedicados para jogos multiplayer, me deparei com um desafio prático: eu precisava que entidades do servidor tivessem comportamentos sofisticados, estruturados e consistentes. Em jogos single-player, esse papel normalmente pertence ao cliente. Em jogos multiplayer modernos, especialmente com servidores de autoridade total, a IA precisa estar no servidor — e isso muda completamente o contexto de performance e arquitetura.

Behavior Trees eram a solução natural, e a biblioteca Mistreevous, criada originalmente em TypeScript por *nikkorn*, tinha o design que eu queria usar: simples, modular, com uma DSL clara e extremamente expressiva. Mas trazê-la para o ecossistema .NET não seria apenas traduzir código. Seria reimaginar sua arquitetura para um ambiente mais exigente, com foco em performance, zero-alocação e previsibilidade.

Este post documenta esse processo.

---

# **Por que Behavior Trees?**

Behavior Trees são amplamente usados em jogos, robótica, simulações e sistemas com agentes. Sua força vem da combinação entre:

* estrutura hierárquica
* leitura clara de intenção
* modularidade forte
* facilidade de depuração
* previsibilidade do fluxo de execução

Eles permitem criar comportamentos complexos a partir de blocos simples.

No contexto de servidores multiplayer, isso significa:

* baixo custo por tick
* comportamento determinístico
* separação entre lógica e estado
* escalabilidade com múltiplas entidades

A Mistreevous já fornecia isso em TypeScript, mas existia um espaço enorme para uma versão em C# que fosse **realmente otimizada para loops de frame**.

---

# **Por que portar a Mistreevous?**

A Mistreevous original possui:

* uma DSL fluida
* nós bem definidos
* parsing simples e fácil de entender
* compatibilidade com JSON
* uma API acessível

Mas ela foi criada para um ecossistema completamente diferente.

Na linguagem C# — especialmente sob .NET 9.0 e .NET Standard 2.1 — há oportunidades de:

* reduzir drasticamente alocações
* otimizar fluxo de execução
* aumentar a eficiência em servidores
* usar estruturas mais adequadas que `Array<T>` de JS
* obter melhor previsibilidade de memória e GC

Se a Mistreevous original era elegante, a versão C# precisava ser elegante **e eficiente em escala**.

---

# **Não foi apenas um port — foi uma reescrita**

Mesmo respeitando a semântica da Mistreevous, a implementação em C# precisou ser reestruturada e otimizada. Abaixo, um resumo do que teve que mudar.

---

# **Zero-allocation: o coração do port**

O objetivo principal era garantir que a execução de `Step()` — equivalente a processar um frame/tick da IA — **não gerasse lixo para o Garbage Collector**.

Em servidores, isso é crítico. Imagine dezenas ou centenas de entidades rodando a IA 20–60 vezes por segundo. Qualquer alocação repetida, mesmo pequena, vira um problema real.

As principais técnicas usadas foram:

### **1. Remoção total de LINQ em hot paths**

LINQ é expressivo, mas cria enumeradores, closures, listas temporárias.

Trechos como:

```csharp
children.Where(x => condition(x))
```

foram substituídos por loops manuais.

---

### **2. Uso intensivo de listas e buffers reutilizáveis**

Buffers thread-safe, pré-alocados, reaproveitados entre ticks.

---

### **3. Parsing manual sem strings temporárias**

Em vez de:

```csharp
var tokens = input.Split(' ');
```

foi aplicado parsing caractere a caractere com Span/Index.

---

### **4. Evitar closures e delegates gerados implicitamente**

Delegates geram capturas que podem alocar.

---

### **5. Iteração por for tradicional**

Evita enumeradores struct/class e toda a maquinaria por trás de `foreach`.

---

### **6. Nós compactos e com poucos objetos**

A versão TS usa objetos para tudo.

A versão C# usa:

* arrays fixos
* ref structs onde aplicável
* objetos leves e sem campos supérfluos

---

# **Compatibilidade total com a DSL e JSON**

Um dos pilares do projeto foi garantir que qualquer árvore definida na Mistreevous original:

```
root {
    sequence {
        action [CheckHealth]
        selector {
            action [Flee]
            action [Fight]
        }
    }
}
```

funcionasse exatamente igual em C#.

Isso envolveu:

* semântica idêntica para SUCCESS, FAILURE e RUNNING
* guards com o mesmo comportamento
* mesma ordenação de execução
* compatibilidade com subtree references
* mesmo fluxo de avaliação para decorators

O objetivo era permitir que desenvolvedores migrassem ou compartilhassem comportamentos entre TS e C#.

---

# **Arquitetura interna do projeto**

A organização do MistreevousSharp foi pensada para refletir tanto a estrutura conceitual da biblioteca quanto facilitar manutenção, extensões e otimizações internas. A seguir está uma visão geral das principais pastas e suas responsabilidades dentro do repositório.

```
MistreevousSharp/
├── assets/                      → Logos, ícones e imagens usadas no README
├── example/                     → Exemplo funcional de uso (MyAgent e Program)
├── src/
│   └── Mistreevous/
│       ├── Agent.cs             → Representa o agente que executa a árvore
│       ├── BehaviourTree.cs     → Núcleo da execução da árvore
│       ├── BehaviourTreeBuilder.cs
│       ├── BehaviourTreeDefinition.cs
│       ├── BehaviourTreeOptions.cs
│       ├── Lookup.cs
│       ├── MDSLDefinitionParser.cs
│       ├── Mistreevous.cs       → API de alto nível (entry point)
│       ├── NodeDetails.cs
│       ├── State.cs
│       ├── Utilities.cs
│
│       ├── Attributes/          → Sistema de atributos e callbacks
│       │   ├── Attribute.cs
│       │   ├── Callbacks/
│       │   │   ├── Callback.cs
│       │   │   ├── Entry.cs
│       │   │   ├── Exit.cs
│       │   │   └── Step.cs
│       │   └── Guards/
│       │       ├── Guard.cs
│       │       ├── GuardPath.cs
│       │       ├── Until.cs
│       │       └── While.cs
│
│       ├── Nodes/               → Hierarquia de nós do behaviour tree
│       │   ├── Node.cs
│       │   ├── Composite.cs
│       │   ├── Decorator.cs
│       │   ├── Leaf.cs
│       │
│       │   ├── Composite/
│       │   │   ├── All.cs
│       │   │   ├── Lotto.cs
│       │   │   ├── Parallel.cs
│       │   │   ├── Race.cs
│       │   │   ├── Selector.cs
│       │   │   └── Sequence.cs
│       │
│       │   ├── Decorator/
│       │   │   ├── Fail.cs
│       │   │   ├── Flip.cs
│       │   │   ├── Repeat.cs
│       │   │   ├── Retry.cs
│       │   │   ├── Root.cs
│       │   │   └── Succeed.cs
│       │
│       │   └── Leaf/
│       │       ├── Action.cs
│       │       ├── Condition.cs
│       │       └── Wait.cs
│
│       ├── Optimizations/       → Utilidades internas focadas em zero-allocation
│       │   └── ZeroAllocationHelpers.cs
│
│       ├── Mistreevous.csproj   → Arquivo do projeto da biblioteca
│       ├── bin/                 → Saída de build
│       └── obj/                 → Arquivos temporários de compilação
│
├── .github/workflows/           → Pipeline de publicação no NuGet
│   └── publish.yml
├── README.md
├── CHANGELOG.md
└── tree.txt                     → Representação visual da estrutura (gerada)
```

---

# **Resumo técnico da arquitetura**

| Pasta / Arquivo             | Responsabilidade                                                               |
| --------------------------- | ------------------------------------------------------------------------------ |
| **src/Mistreevous/**        | Contém toda a API pública e núcleo da biblioteca                               |
| **Nodes/**                  | Implementação da árvore de comportamento — composites, decorators e leaf nodes |
| **Attributes/**             | Sistema de callbacks e guards, compatível com Mistreevous original             |
| **Optimizations/**          | Código crítico voltado a zero-allocation e performance                         |
| **MDSLDefinitionParser.cs** | Parser da DSL original Mistreevous (MDSL)                                      |
| **BehaviourTree.cs**        | Máquina de execução, equivalente ao "engine"                                   |
| **Utilities.cs**            | Funções auxiliares internas                                                    |
| **example/**                | Demonstração de uso completo                                                   |
| **.github/workflows/**      | Pipeline automático de build + NuGet publish                                   |

---

# **Como essa arquitetura se relaciona com a versão TypeScript**

A organização acima foi inspirada na Mistreevous original, porém adaptada para C#:

* Na versão TypeScript, nós são objetos simples; em C#, eles viraram classes dedicadas com foco em performance.
* O parsing da DSL foi reescrito manualmente para manter zero-allocation.
* A hierarquia de arquivos reflete a estrutura conceitual da Mistreevous, facilitando migração e leitura do código original.
* A pasta **Optimizations/** não existe no projeto original — ela centraliza tudo que é específico de .NET e C#.
---

# **Ganho de performance estimado**

Benchmarks simples (sob baixa carga) indicam:

| Operação           | Mistreevous TS | MistreevousSharp | Melhoria            |
| ------------------ | -------------- | ---------------- | ------------------- |
| Parsing da DSL     | ~0.8 ms        | ~0.35 ms         | ~2.3x               |
| Execução por tick  | variável       | estável 0.00x ms | redução substancial |
| Alocações por tick | múltiplas      | zero             | eliminadas          |

O ganho real cresce com o número de entidades.

---

# **Exemplo de uso em C#**

```csharp
var definition = @"
root {
    sequence {
        action [CheckHealth]
        selector {
            action [Flee]
            action [Fight]
        }
    }
}";

var agent = new MyAgent();

var tree = new BehaviourTree(definition, agent);

while (gameRunning)
{
    tree.Step();
}
```

---

# **Aplicações práticas**

Além de servidores multiplayer, a biblioteca é adequada para:

* Unity (sem GC spikes)
* Godot (C#)
* agentes autônomos
* simulações de ambiente
* sistemas embarcados
* robôs e drones com comportamento modular

---

# **Próximos passos**

Estou trabalhando em:

* decorators avançados (Timeout, Cooldown)
* melhorias no parser
* suporte a ScriptableObjects no Unity
* visual editor (C#) para acompanhar a versão TS

---

# **Conclusão**

Portar Mistreevous para C# foi um projeto desafiador e gratificante. Mais do que converter código, foi necessário reinterpretar cada detalhe para o ecossistema .NET, sempre com foco em performance, previsibilidade e compatibilidade.

O resultado é uma biblioteca moderna, leve e preparada para aplicações de alta demanda — mantendo o espírito simples e prático da versão original.

**GitHub:** [https://github.com/guinhx/MistreevousSharp](https://github.com/guinhx/MistreevousSharp)
**NuGet:** [https://www.nuget.org/packages/MistreevousSharp](https://www.nuget.org/packages/MistreevousSharp)
**Original Mistreevous:** [https://github.com/nikkorn/mistreevous](https://github.com/nikkorn/mistreevous)
