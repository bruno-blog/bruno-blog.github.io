---
title: "Executando tarefas agendadas em background em .NET"
description: Como executar tarefas agendadas em background usando a interface IHostedServices
date: 2021-11-28T00:00:00-03:00
author: bruno
tags: [.NET]
categories: [Desenvolvimento]
---

## Como executar tarefas agendadas em background usando a interface IHostedServices

Você já precisou criar algum sistema que processasse em segundo plano? Se a resposta for sim, acredito que você tenha feito uso de frameworks como Hangfire, Quartz.NET ou implementou algo mais rústico à mão para criar windows services e controlar sua execução através do agendador de tarefas do próprio windows não é mesmo? Eu particularmente fiz muito isso (e continuo fazendo hehe) entretanto, atualmente eu crio minhas task de forma mais elegante e minimalista e sem usar frameworks, apenas usando o IHostedServices.

A finalidade deste artigo é demonstrar um modo simples para criação de tasks de execução em segundo plano, e para isso irei usar a interface IHostedServices disponível nas versões do .NET Core (a partir da 2.1) para implementar os serviços.

O código-fonte refatorado está disponível neste link

Neste artigo eu irei focar apenas na implementação, se deseja conhecer melhor os detalhes técnicos acesse a fonte através dos links abaixo:

- [Background tasks with hosted services in ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/host/hosted-services?view=aspnetcore-5.0&tabs=visual-studio)
- [System.Timers](https://docs.microsoft.com/en-us/dotnet/standard/threading/timers)
- [Implementar tarefas em segundo plano em microsserviços com IHostedService e a classe BackgroundService](https://docs.microsoft.com/pt-br/dotnet/architecture/microservices/multi-container-microservice-net-applications/background-tasks-with-ihostedservice)
- [Template Method Pattern from GoF](https://pt.wikipedia.org/wiki/Template_Method)

## Criando o nosso projeto


Primeiro, vamos pavimentar o nosso projeto, abra o seu terminal e execute os comandos abaixo:

```shell
dotnet new console --name CronJobWithDotNetCore
```
Adicione a biblioteca da microsoft que contém o hosts genérico [Microsoft.Extensions.Hosting](https://docs.microsoft.com/pt-br/dotnet/core/extensions/generic-host)
```shell
cd CronJobWithDotNetCore/
dotnet add package Microsoft.Extensions.Hosting
```

Well done!

## Criando a classe BackgroundService

Adicione uma nova classe no projeto (BackgroundService.cs). Essa classe será nossa super clase que servirá como template estendendo seus métodos para seus derivados, irei implementar um estilo de código que segue um padrão chamado [Template Method](https://pt.wikipedia.org/wiki/Template_Method) que por sinal é bem util quando você quer reduzir códigos duplicados, você irá perceber que as classes criadas neste exemplo possuí estruturas parecidas e por isso adotarei este padrão para evitar a duplicação de código aproveitando os benefícios do OPP.

*OBS: Se eu não usar este pattern terei que criar varias classes implementando o mesmo comportamento da classe BackgroundService.cs para executar seus métodos!*

### Conteúdo da classe BackgroundService:

```cs
//A versão refatorada está disponível no github ...
public abstract class BackgroundService : IHostedService, IDisposable
{
    protected BackgroundService(double timeInMiliseconds)
    => _timeInMiliseconds = timeInMiliseconds;

    private System.Timers.Timer _timer;
    private readonly double _timeInMiliseconds;

    protected abstract Task ExecuteAsync(CancellationToken cancellationToken);

    public virtual async Task StartAsync(CancellationToken cancellationToken)
    {
        _timer = new System.Timers.Timer(_timeInMiliseconds);
        _timer.Elapsed += async (sender, arq) => await ExecuteAsync(cancellationToken);
        _timer.Start();
        await Task.CompletedTask;
    }

    public virtual async Task StopAsync(CancellationToken cancellationToken)
    => timer?.Dispose();

    public virtual void Dispose() => timer?.Dispose();
}
```

## Criando nosso primeiro Job

Vamos criar uma classe de serviço que executará um método varias vezes dentro de um intervalo de tempo parametrizável, essa classe representará um job de notificações, e simulará envios de sms e e-mails. Crie um outro arquivo de classe `NotificationJob.cs` e use a super classe `BackgroundService.cs` como herança, em seguida adicione um comportamento no método `ExecuteAsync`.

```cs
//A versão refatorada está disponível no github ...
public class NotificationJob : BackgroundService
{
  public NotificationJob(double timeInMiliseconds) : base(timeInMiliseconds)
  {
  }

  protected override async Task ExecuteAsync(CancellationToken cancellationToken)
  {
        Console.ForegroundColor = ConsoleColor.DarkGreen;
        var time = DateTime.Now;
        Console.WriteLine($"{time} NotificationJob - Send E-Mail");
        Console.WriteLine($"{time} NotificationJob - Send SMS");
        Console.ResetColor();
        await Task.CompletedTask;
  }
}
```

## Modificando a classe Program.cs

Será preciso efetuar uma modificação na classe padrão `Program.cs` que foi gerada automaticamente durante a criação do nosso projeto.

Classe padrão gerada automaticamente:

```cs
class Program
{
  static void Main(string[] args)
  {
      Console.WriteLine("Hello World!");
  }
}
```
Classe padrão modificada.
```cs
public static class Program
{
  public static void Main(string[] args)
    =>  Host.CreateDefaultBuilder(args)
            .ConfigureServices(services => {
               //Register yours services here...
            })
            Build()
            .Run();
}
```

Após efetuar as devidas moficações iremos então registrar a nossa classe de serviço `NotificationJob` em `IHostBuilder.ConfigureServices` com o `AddHostedService` método de extensão:

```cs
//A versão refatorada está disponível no github ...
public static class Program
{
  public static void Main(string[] args)
    =>
    Host.CreateDefaultBuilder(args)
    .ConfigureServices(services =>
    {
        services.AddHostedService<NotificationJob>(_ => new NotificationJob(300000));
    })
    Build()
    .Run();
}
```
Prontinho! resgitramos a nossa classe e passamos via construtor um tempo em milesegundos que representam 5 minutos, agora vamos rodar nossa aplicação para ver o resultado!
```shell
10:00:00 AM NotificationJob - Send E-Mail
10:00:00 AM NotificationJob - Send SMS
10:05:00 AM NotificationJob - Send E-Mail
10:05:00 AM NotificationJob - Send SMS
10:10:00 AM NotificationJob - Send E-Mail
10:10:00 AM NotificationJob - Send SMS
```

\o/ funcionou!

Veja que a primeira execução ocorreu às 10:00, e depois levou um tempo de 5 min para a rotina executar novamente!

E se quiséssemos adicionar mais um job para executar… é possível? Sim, é sim! e irei mostrar isso, continue a leitura!

Executando mais de um Job no mesmo projeto
Agora vamos adicionar mais um job para executar simultanemente a casa 2 minutos, este job irá simular chamadas em uma WebAPI para verificar se ela está de pé, irei seguir os mesmos passos descritos na criação do nosso primeiro job de notificação.

Adicionando uma nova classe ao projeto `HealthCheckJob.cs`:

```cs
public class HealthCheckJob : BackgroundService
{
  public HealthCheckJob(double timeInMiliseconds) : base(timeInMiliseconds)
  {
  }

  protected override async Task ExecuteAsync(CancellationToken cancellationToken)
  {
    Console.ForegroundColor = ConsoleColor.Blue;
    var time = DateTime.Now.ToString("hh:mm:ss");
    Console.WriteLine($"{time} HealthCheckJob - Checking health api.products");
    Console.WriteLine($"{time} HealthCheckJob - Checking health api.customer");
    Console.ResetColor();
    await Task.CompletedTask;
  }
}
```

Registrando o novo job para executar a cada 2 minuto e alterando o tempo de duração do job anterior para 1 minuto:

```cs
  //alterando os milesegundos
  services.AddHostedService<NotificationJob>(_ => new NotificationJob(60000));
  //registrando o novo job
  services.AddHostedService<HealthCheckJob>(_ => new HealthCheckJob(120000));
```

Resultado:

```shell
03:31:28 NotificationJob - Send E-Mail
03:31:28 NotificationJob - Send SMS
03:32:28 NotificationJob - Send E-Mail
03:32:28 NotificationJob - Send SMS
03:32:28 HealthCheckJob - Checking health api.products
03:32:28 HealthCheckJob - Checking health api.customer
03:33:28 NotificationJob - Send E-Mail
03:33:28 NotificationJob - Send SMS
03:34:28 HealthCheckJob - Checking health api.products
03:34:28 HealthCheckJob - Checking health api.customer
03:34:28 NotificationJob - Send E-Mail
03:34:28 NotificationJob - Send SMS
```

Pronto! agora temos 2 jobs intercalados em execução e intervalos diferentes no mesmo projeto.

Curtiu? Espero que tenham gostado deste conteúdo. E me diz ai, como se chama os serviços que são executados em background em sua empresa? Job, robô, rotina, schedule, tasks?

Até a proxima.