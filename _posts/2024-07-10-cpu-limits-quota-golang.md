---
title: "Golang - Container e CPU Limits quota"
description: Definir limites de CPU sem configurar GOMAXPROCS pode não ser uma boa ideia.
date: 2024-07-1-T00:00:00-03:00
author: bruno
tags: [golang, container]
categories: [Desenvolvimento]
---

Recentemente, me deparei com um cenário em produção um tanto curioso. Ocorreu que, durante o monitoramento de uma aplicação, percebemos que ocorreram eventos de throttling de CPU e, coincidentemente, nesse período houve um aumento significativo da latência, seguido de um aumento do tempo de parada do GC (garbage collector). Bem, neste artigo compartilho como resolvemos esse evento e sua causa raiz.

![img](/assets//img/th.png){: width="700" height="400" }
*Throttling de CPU que ocorria eventualmente em cenários onde havia muito processo em execução*

## O problema 

Conforme relatei no início deste artigo, durante o monitoramento de uma aplicação notamos um aumento significativo no tempo de parada do GC, latência e throttling de CPU em nosso ambiente produtivo (em nossas máquinas isso não ocorria hehe) e depois de alguns testes notamos que se retirássemos os valores de limit resources aplicados no arquivo de configuração do Kubernetes esse evento parou de acontecer. Foi daí que conseguimos entender que existia algo estranho no comportamento entre o Golang e o CFS do Linux quando definíamos algum limit de CPU. Navegando na internet nos deparamos com diversos cases bacanas que serviram como inspiração para a criação deste artigo. No final deste artigo deixarei as referências

> Para este artigo não ficar muito extenso, não irei abordar sobre o CFS e sua relação com CPU Limits (embora ele seja uma parte importante).

## Breve explicação sobre CPU Limits e CFS

Os limites de CPU éuma parte específica do controle oferecido pelos cgroups. Eles permitem definir quanto tempo de CPU um grupo de processos pode utilizar durante um período determinado. Isso garante que processos importantes tenham acesso prioritário à CPU evitando a monopolização de outros processos. Dessa forma, podemos especificar uma porcentagem da capacidade total de uma CPU; por exemplo, configurar um limite de 50% para um recurso significa que os processos desse recurso só poderão utilizar metade da capacidade da CPU, mesmo que haja mais recursos disponíveis e ociosos.

"Mas qual é a relação disso com o meu problema? Tudo!"

Ocorre que, enquanto os CPU limits consideram os cgroups, o runtime do Golang olha diretamente para os recursos do da máquina host, ignorando completamente as limitações impostas pelos CPU limits afetando a distribuição do tempo que é gerenciado pelo CFS, desse modo se você tiver uma màquina com 16 nucleos o runtime do Golang criará 16 thereads do SO independente dos limits de CPPU aplicadas no cgroup.

## Simulando este evento localmente

Abaixo eu irei demostrar através de uma simulação em minha maquina local este comportamento.

>Para o exemplo abaixo estou rodando uma aplicação GO num container com 2 vCPU e 4GB de RAM.

1. Neste primeiro cenário temos uma aplicação rodando em um container sem NENHUM limit resource aplicado. Podemos observar que a aplicação está executando normalmente sem nenhum problema aparente (Se você não está familiarizado com essa interface veja este [Post]() que eu criei onde explico o que cada item representa).

    Legenda:
    - Proc 0 e 1 representam as 2 threads da maquina host que o Golang utilizou para agendar a execução das goroutines.
    - Cores Azul e Rosa representam as corotinas (goroutines) sendo executadas sob as threads do SO.
    - Tarja bege abaixo das tarjas rosa e azul representam algumas chamadas syscall que a aplicação efetua para efeito de demonstração.

    ![figura-1](/assets/img/figure-1.png)
    *Funcionamento normal*


2. Agora, vejamos o que acontece quando aplicamos um limit de cpu, neste caso usarei o seguinte comando: 

    `docker run --cpus 0.5 ...` (este comando limitará o uso de 50% da cpu)

    Perceba que agora temos um intervalo de espaço maior entre as cores beges e se expandir a img conseguirá notar que a timeline em alguns casos ultrapassam os 50ms e que as tarjas beges executam em um periodo de 25ms e o restante desse tempo a goroutine esta parada fazendo nada e aqui está o throttling que mencionei!
    ![figura-2](/assets/img/figure-2.png)
    *Funcionamento com limite de 0.5 de CPU aplicado no container*

    Como já mencionado isto ocorreu porque o runtime do Golang ignorou a configuração de limits de cpu aplicada no container, o ideal seria ele se basear nos valores especificados no cgroups (limits.cpu) e utilizar 1 núcleo de CPU ao invez de 2, mas para resolver isto é simples, basta especificar manualmente a quantidade de CPU para o runtime do golang atráves da váriavel `GOMAXPROCS`

    >A variável GOMAXPROCS limita o número de threads do sistema operacional que podem executar código Go de nível de usuário simultaneamente. Não há limite para o número de threads que podem ser bloqueados em chamadas de sistema em nome do código Go; elas não contam para o limite GOMAXPROCS. A função GOMAXPROCS deste pacote consulta e altera o limite. [documentação oficial](https://pkg.go.dev/runtime)
    

3. Por ultimo, executei o mesmo container mas agora estou especificando a quantidade de CPU para o runtime do Golang atráves da váriavel `GOMAXPROCS`, este é o comando: `docker run -e GOMAXPROCS=1 --cpus 0.5` vejamos o resultado abaixo:
   ![figura-3](/assets/img/figure-3.png)
   *Funcionamento com limite de 0.5 de CPU aplicado no container e GOMAXPROCS=1*

    Como podemos ver, agora a aplicação está sendo executada a cada 50ms sobe 1 thread do SO em seguida, não realiza nenhuma execução durante os 50 ms, exatamente como podemos esperar para esta cota.


# Conclusão

Se você executa uma aplicação Go dentro de um container com limites de CPU estabelecido é importante especificar esses limites para o runtime do Go saber que essas limitações foram aplicadas, atualmente existem alguns pacotes que fazem isto, um que conheço e utilizo é o [automaxprocs](https://github.com/uber-go/automaxprocs) que foi contruído e mantído pelo time de desenvolvimento da Uber.

Esse foi o resultado que obtivemos ao utilizar o pacote [automaxprocs](https://github.com/uber-go/automaxprocs), basicamente o que ele irá fazer é configurar a váriavel GOMAXPROCS automaticamente em tempo de execução e tentará redimensionar isso conforme as limitações do csgroups.

![fim](/assets/img/th2.png)
*A linha verde representa o ANTES, enquanto a parte amarela representa o DEPOIS do uso da lib automaxprocs*

uma outra maneira de se configurar isto é setaando o `limits.cpu` no arquivo de configuração do kubernetes:

```yaml
env:
- name: GOMAXPROCS
  valueFrom:
    resourceFieldRef:
    resource: limits.cpu
```

# Referência:

Segue as referencias que serviram como o guia para entendimento e resolução deste evento:

1. [kubernetes-cpu-limits-go](https://www.ardanlabs.com/blog/2024/02/kubernetes-cpu-limits-go.html)

2. [cgroup-throttling](https://danluu.com/cgroup-throttling/)

3. [stop-using-cpu-limits](https://home.robusta.dev/blog/stop-using-cpu-limits)
