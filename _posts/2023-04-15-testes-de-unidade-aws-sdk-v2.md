---
title: "Testes de unidade utilizando aws-sdk-go-v2"
description: Veja como implementar testes de unidade utilizando a biblioteca aws-sdk-go-v2 na linguagem GO
author: bruno
date: 2023-04-15T00:00:00-03:00
categories: [Desenvolvimento]
tags: [Golang]
math: true
mermaid: true
---


Veja como implementar testes de unidade utilizando a biblioteca [aws-sdk-go-v2]("https://github.com/aws/aws-sdk-go-v2") na linguagem GO.

## AWS SDK

Em resumo, o AWS SDK é um pacote que permite interagir com os serviços da AWS (Amazon Web Services) usando uma linguagem de programação. Este SDK foi projetado para simplificar o desenvolvimento de aplicações que utilizam serviços da AWS, fornecendo uma interface de programação fácil de usar.

O AWS SDK for Go tem duas versões principais: *v1* e *v2*. Basicamente na *v2* teremos a evolução do sdk e melhorias de algumas deficiências da versão anterior (*v1*), a *v2* foi construída em torno de uma abordagem mais moderna baseada em contextos e utilizando recursos mais recentes da linguagem Go além de aprimorar o processo de configuração e autenticação.

Na **versão 1** do sdk tínhamos acesso a uma **interface** de cada serviço da aws (conhecida como *iface, dynamoiface, s3iface, etc...) e isso facilitava a implementação de **mocks** para os testes de unidade, entretanto dependendo do caso de uso tinha o risco de criarmos **mocks** para toda a interface, na versão atual essa interface deixou de existir e então precisamos criar as nossas próprias **interface** (ficou bem melhor dessa forma na minha opinião).

## Show me the code!

Antes, vamos entender o problema que iremos resolver. O diagrama abaixo representa a jornada do nosso app de exemplo.
O nosso app validará se um metadata vindo do SQS é valido e persistirá ele no DynamoDB.

```
       ┌────┐                                 
       │Main│                                 
       └──┬─┘                                 
 ┌────────▽───────┐                           
 │Metadata Service│                           
 └────────┬───────┘                           
  ┌───────▽───────┐                           
  │Receive Message│                           
  │From SQS       │                           
  └───────┬───────┘                           
  ________▽________                           
 ╱                 ╲             ┌───┐        
╱ Metadata Is Valid ╲____________│Yes│        
╲                   ╱yes         └─┬─┘        
 ╲_________________╱    ┌──────────▽─────────┐
          │no           │Put Item in DynamoDB│
  ┌───────▽──────┐      └────────────────────┘
  │Printf message│                            
  └──────────────┘                            
```

#### DynamoDB:

```go
// /infra/db.go
package infra

import (...)

type DynamoAPI interface {
  PutItem(ctx context.Context, params *dynamodb.PutItemInput, optFns ...func(*dynamodb.Options)) (*dynamodb.PutItemOutput, error)
}

type db struct {
  api DynamoAPI
}

func (i *db) PutItem(ctx context.Context, params *dynamodb.PutItemInput, optFns ...func(*dynamodb.Options)) (*dynamodb.PutItemOutput, error) {
  return i.api.PutItem(ctx, params, optFns...)
}

func NewDatabase(client *dynamodb.Client) DynamoAPI {
  return &db{
    api: client,
  }
}

```


#### SQS:
```go
// /infra/queue.go
package infra

import (...)

type SqsAPI interface {
  ReceiveMessage(ctx context.Context, params *sqs.ReceiveMessageInput, optFns ...func(*sqs.Options)) (*sqs.ReceiveMessageOutput, error)
}

type queue struct {
  api SQSAPI
}

func (sqs *queue) ReceiveMessage(ctx context.Context, params *sqs.ReceiveMessageInput, optFns ...func(*sqs.Options)) (*sqs.ReceiveMessageOutput, error) {
  return sqs.api.ReceiveMessage(ctx, params, optFns...)
}

func NewQueueInfra(client *sqs.Client) SQSAPI {
  return &queue{
    api: client,
  }
}

```

#### Criando a struct de serviço:

```go
package service

import ( ... )

type MetadataInput struct {
  Id        string `json:"id"`
  Value     string `json:"content"`
  CreatedAt string `json:"createdAt"`
}

type metadataService struct {
  database infra.DynamoAPI
  queue    infra.SQSAPI
}

func (service *metadataService) Process(ctx context.Context) error {

  output, err := service.queue.ReceiveMessage(ctx, &sqs.ReceiveMessageInput{
    QueueUrl: aws.String("sqs-url-name"),
  })

  if err != nil {
    return err
  }

  for _, msg := range output.Messages {

    var input MetadataInput
    if err := json.Unmarshal([]byte(*msg.Body), &input); err != nil {
      log.Println(err.Error())
      continue
    }

    if strings.TrimSpace(input.CreatedAt) == "" {
      log.Println("the createdAt field is invalid")
      continue
    }

    item, err := attributevalue.MarshalMap(input)
    if err != nil {
      log.Println(err.Error())
      continue
    }

    _, err = service.database.PutItem(ctx, &dynamodb.PutItemInput{
      Item:      item,
      TableName: aws.String("table-name"),
    })

    if err != nil {
      log.Println(err.Error())
      continue
    }
  }

  return nil
}

```


#### Escrevendo os mocks para os testes:


#### Dynamo Mock:

```go
type dynamoMock struct {
  PutItemFnMock func() (*dynamodb.PutItemOutput, error)
}

func (m *dynamoMock) PutItem(ctx context.Context, params *dynamodb.PutItemInput, optFns ...func(*dynamodb.Options)) (*dynamodb.PutItemOutput, error) {
  return m.PutItemFnMock()
}

```

#### SQS Mock:


```go
type sqsMock struct {
  ReceiveMessageFnMock func() (*sqs.ReceiveMessageOutput, error)
}

func (sqs *sqsMock) ReceiveMessage(ctx context.Context, params *sqs.ReceiveMessageInput, optFns ...func(*sqs.Options)) (*sqs.ReceiveMessageOutput, error) {
  return sqs.ReceiveMessageFnMock()
}

```

#### Escrevendo os testes:

1. No teste abaixo o comportamento esperado é que o método `Proccess` termine se receber algum erro vindo do SQS, para isto irei simular tal comportamento atraves do mock.

```go
  t.Run("It should return sqs error when there is", func(t *testing.T) {

    service := metadataService{
      queue: &sqsMock{
        ReceiveMessageFnMock: func() (*sqs.ReceiveMessageOutput, error) {
          return nil, fmt.Errorf("failed")
        },
      },
    }

    gotErr := service.Proccess(context.TODO())
    assert.NotNil(t, gotErr)
    assert.Equal(t,"failed", gotErr.Error())

  })

```

2. O próximo teste tem por finalidade garantir que uma mensagem valida seja persistida no dynamo.

```go
  t.Run("It should receive a msg from sqs and successfully persist", func(t *testing.T) {

    service := metadataService{
      database: &dynamoMock{
        PutItemFnMock: func() (*dynamodb.PutItemOutput, error) {
          return nil, nil
        },
      },
      queue: &sqsMock{ReceiveMessageFnMock: func() (*sqs.ReceiveMessageOutput, error) {
        return &sqs.ReceiveMessageOutput{
          Messages: []types.Message{
            {
              Body: aws.String(string([]byte(`{"id":"dummy","content":"dummy","createdAt":"2023-04-02T15:04:05Z07:00"}`))),
            },
          },
        }, nil
      }},
    }

    gotErr := service.Proccess(context.TODO())
    assert.Nil(t, gotErr)

  })
```

##### F I M


Clique [aqui](https://github.com/brbarmex/golang-unit-test-aws-sdk-v2) para ter acesso a versão completa no github.





