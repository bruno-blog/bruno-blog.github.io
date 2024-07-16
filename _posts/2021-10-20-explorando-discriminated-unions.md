---
title: "Explorando Discriminated Unions em C# com o Pacote OneOf"
description: Entendendo Discriminated Unions em C# com o Pacote OneOf
date: 2021-11-28T00:00:00-03:00
author: bruno
tags: [.NET]
categories: [Desenvolvimento]
---

## Explorando Discriminated Unions em C# com o Pacote OneOf

Em programação C#, os Discriminated Unions é uma maneira de lidar com diferentes tipos de dados de forma organizada e segura permitindo representar casos diferentes de dados em uma estrutura única, o que pode ser muito útil em muitas situações.

### O que são Discriminated Unions?
Imagine que você tem uma estrutura de dados que pode ter diferentes formatos como resposta (return). Por exemplo, você pode ter um resultado que pode ser um sucesso com um número inteiro ou um erro com uma mensagem. Discriminated Unions ajudam a lidar com esses casos de forma clara.

### Como Implementar com OneOf

Vamos ver como usar o pacote OneOf para implementar Discriminated Unions em C# de maneira simples e eficaz.

1. Instalando o Pacote OneOf:

```shell
dotnet add package OneOf
```

1. Criando a classe:

```cs
using OneOf;

public class Resultado : OneOfBase<int, string>
{
    private Resultado(OneOf<int, string> input) : base(input) { }

    public static Resultado CriarSucesso(int valor)
    {
        return new Resultado(valor);
    }

    public static Resultado CriarErro(string mensagem)
    {
        return new Resultado(mensagem);
    }
}

```

1. Resultado:
```cs
public class Programa
{
    public static void Principal()
    {
        var resultado = ObterResultado(42);
        
        if (resultado.IsT0)
        {
            Console.WriteLine($"Sucesso: {resultado.AsT0}");
        }
        else if (resultado.IsT1)
        {
            Console.WriteLine($"Erro: {resultado.AsT1}");
        }
    }
    
    public static Resultado ObterResultado(int valor)
    {
        if (valor > 0)
        {
            return Resultado.CriarSucesso(valor);
        }
        else
        {
            return Resultado.CriarErro("O valor não pode ser negativo ou zero.");
        }
    }
}
```

Exemplo de um caso de uso: [clique_aqui](https://github.com/brbarmex/dotnet-oneof-sample/tree/main)


