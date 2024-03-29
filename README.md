# OIC - Caso de Uso 1 - Performance com o Oracle ERP Cloud

O Objetivo deste documento é demonstrar um caso de uso típico de implementação em OIC não-performática e quais são as alternativas possíveis para torná-la mais produtiva.

Este caso de uso vai lhe ajudar caso esteja pensando em:
* Uso de Banco de Dados para consultas e processamento em Loops
* Chamadas à API REST/SOAP da aplicação Oracle (Oracle ERP Cloud entre outras)
* Criar uma rotina complexa e com volume grande de dados

-----
# T1 - Cenário Atual (As-Is)

Na Figura abaixo, temos um caso de processamento de boletos em atraso, no qual existe um botão que dispara o processamento. O botão e parte deste processamento está fora do OIC, porém, nada impede que o caso de uso poderia ter ou não esta parte dentro do OIC.

Vamos abstrair os detalhes deste processamento, imaginando que, um usuário clica no botão e espera que os boletos em atraso dos últimos 5 anos possam ser renegociados, impressos ou algum tipo de processamento adicional seja realizado neste momento.
Qualquer processamento adicional pode estar presente ou não neste caso de uso. O importante aqui é analisar a situação dos objetivos de forma bem abstrata. Logo, este caso de uso pode ser útil para várias outras situações.

Após o clique do botão, é executada uma consulta em banco de dados (Query) no qual é realizada sobre todos os dados de boletos dos últimos 5 anos.

![Fig 1](https://github.com/hoshikawa2/OIC-CASE-1/blob/master/Images/Fig1.jpg?raw=true)

    Aqui vale ressaltar que o ideal para casos de uso como este, seria utilizar
    mecanismos assíncronos, no qual se possa utilizar de paralelismos de processamento
    e notificação. Associar um botão para processamento síncrono é um risco grande para
    o sucesso da operação pois em alguma etapa deste caso de uso pode haver travamentos
    como dead-locks em banco de dados, falha de comunicação (considerando front-end x 
    back-end).
    Considere também utilizar recursos nativos das aplicações Oracle como processamento
    em batch nas APIs REST/SOAP, pois a maior parte dos serviços possuem esses mecanismos.

**T1.1 Análise da Query em Banco de Dados**

Neste caso de uso, trata-se de uma consulta extremamente demorada, pois envolve as tabelas de Boletos e suas respectivas parcelas dos últimos 5 anos. Possivelmente, ainda existem complicadores como dados que não estão tratados para uso imediato do processamento em questão e que podem demandar algum tipo de tratamento durante a execução deste caso de uso.

Vamos levar em consideração que esta query traz uma grande quantidade de dados causando um custo grande de processamento no banco de dados e também trazendo um grande volume de dados através da comunicação em rede.

**T1.2 Análise: Loop com chamadas a serviços externos**

Logo após a execução da **Query (T1.1)**, percebe-se que existe um **Loop** o qual irá realizar várias chamadas a uma API externa (**ERP SaaS**) através do **OIC**.

São previstas 1.000 chamadas a esta API o que invariavelmente irá causa um excesso de chamadas à API de origem no **ERP SaaS** com uma sobrecarga e delay para completar o processamento.
Isto ocorre porque o **OIC** é capaz de receber as requisições em um tempo X e faz a requisição ao **ERP SaaS** capaz de atender a um tempo Y. O tempo X é menor que o tempo Y, causando várias chamadas em espera durante um certo tempo.
O risco aqui é a abertura de inúmeras threads em espera para completar o processamento todo, o que pode acarretar mais demora no processamento, além de possibilitar a perda de comunicação e causar danos à transação toda.

Portanto evite ao máximo situações como esta.


----
# T2 - Possíveis Soluções

**T2.1 Consultas a bancos de dados**

Caso clássico de consulta a um banco de dados para que em seguida, possamos utilizar as linhas obtidas para processamento.
Existem vários artigos para otimização de banco de dados e eles também valem aqui.

Procurar executar queries enxutas, que tragam apenas as linhas e as colunas que serão úteis para o processamento. Qualquer coisa fora deste contexto, se torna inútil, custoso e lento.

Algumas observações típicas para otimização de queries são:


    Criar um índice de banco de dados pode ajudar na performance da execução desta consulta.
    Não utilizar muitos JOINS na query, pois isto envolve muito processamento de disco e memória.
    VIEWS são mais rápidas que uma execução de SELECT pois são compiladas no banco de dados.
    EVITE a todo custo expressões com LIKE/% ou IN. Índices não vão ajudar nestes casos.    

    Além disto, talvez valha a pena a criação de uma stored procedure, em casos de necessidade
    de preparação das informações, para que a consulta esteja compilada no banco de dados e possa 
    ser executada de forma imediata. Isto vai ajudar bastante na performance.

    A procedure também pode ser considerada para casos em que se possa tratar os dados 
    para otimizar mais ainda o processamento do caso de uso. Muitas vezes, não é 
    possível resolver numa query só estes problemas. 


**T2.2 Substituir Consulta ou Ação Única por Lotes**

Dentro do OIC (na figura abaixo, grifado em azul) são executadas inúmeras chamadas (1.000x) para a API do **ERP SaaS** ocasionando o efeito analisado em **T1.2**.


![Fig 8](https://github.com/hoshikawa2/OIC-CASE-1/blob/master/Images/Fig8.jpg?raw=true)

Uma boa prática neste cenário é evitar que **Loops** façam chamadas se for excessivamente custoso para o OIC.

Mas como saber quando o número de chamadas for excessivo????

Leve em consideração os seguintes pontos para avaliar quando usar ou não **Loops**:

    Meça o tempo das chamadas (a APIs, queries de banco de dados, etc) dentro deste Loop
    Procure executar com a quantidade de dados mais próximo do real
    
    Está considerando Loop dentro de Loop? Lembre-se que o tempo é multiplicado pelo número de passos dentro deste Loop. Logo, use Loops com inteligência

**Loops** são extremamente úteis, portanto não descarte usá-los, porém leve em consideração o tempo **TOTAL** de execução e faça testes mais próximos da realidade para avaliar considerar ou não seu uso.

    Você pode considerar usar Loops dentro do OIC quando estiver pensando em execuções em BATCH (temporizadas)


Na figura anterior, podemos notar (conforme as observações feitas) que existe um **Loop** e dentro dele há inúmeras chamadas a **API** do **ERP SaaS**, causando uma demora entre uma requisição e outra. Uma outra forma de tratar isto, sabendo que não será possível evitar o Loop, é primeiro realizar esta chamada a API de forma que não seja necessário executar inúmeras vezes e sim chamá-la uma única vez, em lote.
Repare que podemos considerar que a chamada a **API** do **ERP SaaS** pode ser uma consulta ou uma outra requisição do tipo UPDATE, por exemplo. Em ambos os casos, é válido tentar uma execução em **Lote**.

Quando consideramos utilizar **APIs** do **ERP SaaS**, lembre-se que existem várias formas de tratar consultas e processamentos em **Lote** o que é uma **boa prática**.

    Aplicações SaaS consideram utilizar processamento em Lotes como boa prática
    Isto porque a Cloud traz uma série de benefícios, mas se comparado com o mundo on-premisses, 
    temos que levar em consideração uma latência maior com a que estávamos acostumados 
    quando implementamos em um ambiente on-premisses.

Logo, uma boa forma de fazer isto antes de continuar o processamento é tentar encontrar uma maneira de executar em lote como abaixo:

![Fig 2](https://github.com/hoshikawa2/OIC-CASE-1/blob/master/Images/Fig2.jpg?raw=true)

Mesmo que não seja possível evitar o **Loop** fica mais leve depois utilizar as informações capturadas anteriormente em uma única consulta.

**T2.3 Agendamento**

Processamentos particionados e agendados também podem ser a solução no lugar de tentar processar tudo sequencialmente e de uma única vez.
Muitas vezes, e por incrível que pareça, um processo não necessita de resposta imediata porque o objetivo do negócio não requer isto.
Demandas que necessitam de processamento e resposta próximo ao **tempo-real** são aquelas transações em que o usuário ou cliente realmente precisa disto para continuar seu trabalho ou atividade. Leve isto em consideração.

Talvez seja mais fácil fazer uma preparação das informações durante a madrugada
para que no início do dia, tudo possa ser processado de forma mais leve

Em nosso caso de uso, o processo original implica em um botão que dispara o processamento dos boletos em atraso.
Levando em consideração as análises anteriores em **T1**, uma boa abordagem seria deixar o processamento mais leve fazendo uma preparação prévia das informações durante a madrugada.

    No caso da query inicial para buscar os boletos em atraso dos últimos 5 anos e suas parcelas, 
    é realmente necessário que esta consulta seja realizada no momento em que o usuário pressiona o botão? 
    Ou podemos preparar estes dados na madrugada anterior? 
    As informações mudam constantemente para impedir que a preparação seja feita momentos antes?

O que isto quer dizer?

Vimos que possivelmente, uma query para trazer todos os boletos em atraso e suas parcelas dos últimos 5 anos pode causar uma certa demora para consultar no banco de dados.
Vimos também que possivelmente, ainda haja algum tipo de processamento adicional para executar as tarefas pretendidas no caso de uso.

Com base nisto, poderíamos propor processamentos agendados para preparação de um banco de dados mais leve:


    Considerar a criação de um banco de dados enxuto com o objetivo de otimizar
    ao máximo o processamento para o caso de uso
    
    Considerar otimizar a estruturação das tabelas a fim de atender a demanda
    
    Considerar fazer updates incrementais para otimizar o tempo de processamento
    agendado
    
    Considerar fazer o PURGE de dados e atualizações (exemplo: boletos já pagos
    necessitam ser apagados ou atualizados)

![Fig 3](https://github.com/hoshikawa2/OIC-CASE-1/blob/master/Images/Fig3.jpg?raw=true)

![Fig 4](https://github.com/hoshikawa2/OIC-CASE-1/blob/master/Images/Fig4.jpg?raw=true)

![Fig 5](https://github.com/hoshikawa2/OIC-CASE-1/blob/master/Images/Fig5.jpg?raw=true)

![Fig 6](https://github.com/hoshikawa2/OIC-CASE-1/blob/master/Images/Fig6.jpg?raw=true)

**T2.4 Load-Balancer**

Na pior das hipóteses, se de maneira alguma, não for possível uma abordagem de lote e a performance seja algo realmente crítico, podemos adotar uma implementação com chamada atômica através de vários OICs com Load-Balancer.
Uma ressalva é de que o OIC é dependente dos endpoints que ele chama, portanto, não vai adiantar nada a clusterização de OICs se os endpoints da implementação não estiverem no mesmo formato. Os endpoints acabarão virando o "gargalo" da integração.

![Fig 11](https://github.com/hoshikawa2/OIC-CASE-1/blob/master/Images/Fig11.jpg?raw=true)


**T2.5 Processamento Síncrono/Assíncrono e Filas de Mensagens**

Um **caso típico** onde existe problema de processamento e demora ocorre quando existe uma sequencia de processamento a ser feito e uma tarefa acaba dependendo de outra, logo, é necessário aguardar a execução da anterior para prosseguir.

Mas e se as tarefas não necessitarem desta espera?

Podemos considerar neste caso que existem alternativas para processar as atividades em paralelo, tornando o tempo de resposta bem mais rápido.
Para isto, o OIC conta com o uso de **PUBLISH/SUBSCRIBER**, conhecido também como fila de mensagens.

    Considere os pontos em que sua aplicação possa contar com este recurso pois 
    nestes casos, é possível liberar sua aplicação de continuar o processamento
    (muitas vezes liberando o usuário para continuar suas atividades) e 
    enviando a resposta de forma assíncrona após a realização de todas as atividades
    de processamento

![Fig 7](https://github.com/hoshikawa2/OIC-CASE-1/blob/master/Images/Fig7.jpg?raw=true)

----

# T3 - Materiais de Ajuda

**Armadilhas do padrão de integração comum e práticas recomendadas de design**

https://docs.oracle.com/en/cloud/paas/integration-cloud/integrations-user/common-integration-pattern-pitfalls-and-design-best-practices.html#GUID-09CEC808-3110-4EE4-9478-666A17451458

**Oracle ERP Cloud - APIs de Consultas em Lote com filtros e colunas**

https://docs.oracle.com/en/cloud/saas/financials/20b/farfa/op-payablespayments-get.html

![Fig 9](https://github.com/hoshikawa2/OIC-CASE-1/blob/master/Images/Fig9.png?raw=true)

**Oracle ERP Cloud - Ações em Lote**

https://docs.oracle.com/en/cloud/saas/financials/20b/farfa/Batch_Actions.html

https://docs.oracle.com/en/cloud/saas/procurement/20d/fapra/Batch_Actions.html

![Fig 10](https://github.com/hoshikawa2/OIC-CASE-1/blob/master/Images/Fig10.png?raw=true)

**Publish/Subscriber (Queue/Filas)**

https://www.techsupper.com/2017/07/integration-to-publish-messages-to.html

https://blogs.oracle.com/integration/integration-patterns-publishsubscribe-part1

https://blogs.oracle.com/integration/integration-patterns-publishsubscribe-part2

**Agendamentos no OIC**

https://docs.oracle.com/en/cloud/paas/integration-cloud/integrations-user/creating-scheduled-integrations.html#GUID-9632A5C8-98A7-4371-B542-6A8583427C8D

**Database Query no OIC**

https://docs.oracle.com/en/cloud/paas/integration-cloud/database-adapter/using-oracle-database-adapter-oracle-integration.pdf
https://www.techsupper.com/2017/11/bind-query-parameter-in-database.html

**Melhorar o desempenho dos fluxos de integração Oracle que usam chamadas REST**

https://www.ateam-oracle.com/improving-the-performance-of-oracle-integration-flows-that-use-rest-calls

----
# T4 - Conclusão

Escrever uma aplicação pode ser bastante complexo, muitas vezes o fazemos com um cenário em mente porém este muda drasticamente trazendo um outro mais complexo, com mais dados a serem processados, trazendo muitas vezes problemas de tempo de resposta ou até o colapso geral da aplicação.

O cenário e as soluções apresentadas aqui representam uma pequena parte dos problemas que acontecem em geral, porém a partir daqui podemos avaliar vários outros casos de uso e complementar o que for necessário para uma solução efetiva de implementação com performance e custo baixo.

Se você leu este documento até aqui, por favor, contribua com sugestões de novos casos de uso ou soluções.

Obrigado!
