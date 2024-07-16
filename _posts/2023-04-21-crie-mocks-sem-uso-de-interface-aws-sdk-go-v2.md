---
layout: post
title: "Criando mocks sem utilizar interfaces (Trabalhando com aws-sdk-go-v2)"
description: Aprenda uma maneira diferente de criar mocks para testes de unidade quando utilizar o pacote aws-sdk-go-v2.
date: 2023-04-21T00:00:00-03:00
author: bruno
tags: [Golang]
categories: [Desenvolvimento]
---

Aprenda uma maneira diferente de criar `mocks` para testes de unidade sem a necessidade de usar uma `struct` fake que implementa uma interface com funções de `clients` do pacote [aws-sdk-go-v2]("https://github.com/aws/aws-sdk-go-v2").

Mock é uma técnica na programação onde criamos estruturas para simular o comportamento de um objeto ou algum componente real durante os testes automatizados, mas sem depender desses componentes externos ou até mesmo de interações complexas com o mundo real.

Há várias formas de gerar mocks para testes automatizados. Uma das maneiras mais comuns é utilizar objetos que implementem contratos de interfaces e substituir o comportamento desses objetos conforme a bateria de testes é executada. 

Recomento a leitura deste [post](./2023-04-21-crie-mocks-sem-uso-de-interface-aws-sdk-go-v2.md) onde eu explico sobre como implementar testes de unidade utilizando interfaces para "mockar" comportamentos do [aws-sdk-go-v2]("https://github.com/aws/aws-sdk-go-v2").

## Show me the code!

Será usado o mesmo exemplo descrito neste [artigo](./2023-04-21-crie-mocks-sem-uso-de-interface-aws-sdk-go-v2.md), porém, não criaremos nenhuma struct com funções simuladas de interfaces, em vez disso utilizaremos o middleware do pacote aws-asdk-go-v2. O middleware do pacote aws-sdk-go-v2 é um componente que permite adicionar comportamentos personalizados ao pipeline de solicitação de serviços da AWS. Ele fornece uma interface para manipular solicitações e respostas, e suporta vários recursos, como interceptores, validadores e registradores de solicitação.

*Para saber mais visite a documentação oficial da AWS clicando [aqui](https://aws.github.io/aws-sdk-go-v2/docs/middleware/)*

#### Criando um middleware personalizado para o SQS

```go
/*
      Essa funcao irá retornar como resposta as informacoes inseridas no parametro.
*/
func receiveMessageMock(testeId string, outputExpected *sqs.ReceiveMessageOutput, errExpected error) func(*middleware.Stack) error {
    return func(stack *middleware.Stack) error {
        return stack.Finalize.Add(
            middleware.FinalizeMiddlewareFunc(
                testeId,
                func(ctx context.Context, in middleware.FinalizeInput, next middleware.FinalizeHandler) (out middleware.FinalizeOutput, md middleware.Metadata, err error) {
                    out.Result = outputExpected
                    return out, md, errExpected
                },
            ),
            middleware.Before,
        )
    }
}
```

#### Exemplo de uso nos testes

```go

func TestWithoutUseInterface(t *testing.T){
   
   /// ARRANGE

    ctx := context.TODO()

    outputExpected := &sqs.ReceiveMessageOutput{
        Messages: []sqsTypes.Message{
            {
                Body: aws.String(string([]byte(`{
                    "id":"dummy",
                    "content":"dummy",
                    "createdAt":"2023-04-02T15:04:05Z07:00"}`))),
            },
        },
    }

    cfg, _ := config.LoadDefaultConfig(ctx,
        config.WithRegion("sa-east-1"),
        config.WithAPIOptions([]func(*middleware.Stack) error{
          receiveMessageMock("abc", outputExpected, nil) // <-- A MAGICA ACONTECE AQUI
        }), 
    )

    sqsClient := sqs.NewFromConfig(cfg)

    // ACT

    output, _ := sqsClient.ReceiveMessage(ctx, &sqs.ReceiveMessageInput{
        QueueUrl: aws.String("brbarmex-sqs"),
    })

    // ASSERT

    if output == nil {
        t.Fatal()
    }

}

```

## Considerações

Este é um exemplo simples que pode ser usado como alternativa ao uso de interfaces. Não há nada de errado em utilizar interfaces, mas se você as utiliza apenas para facilitar seus testes, essa abordagem pode ajudar a preservar o design.

