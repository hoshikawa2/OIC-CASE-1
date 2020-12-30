# OIC-CASE-1

O Objetivo deste documento é demonstrar um caso de uso típico de implementação em OIC não-performática e quais são as alternativas possíveis para torná-la mais produtiva.

-----
# T1 - Cenário Atual (As-Is)

Na Figura abaixo, temos um processo de processamento de boletos em atraso, no qual existe um botão que dispara o processamento.
Após o clique do botão, é executada uma consulta em banco de dados (Query) no qual é realizada sobre todos os dados de boletos dos últimos 5 anos. 

![Fig 1](https://github.com/hoshikawa2/OIC-CASE-1/blob/master/Images/Fig1.jpg?raw=true)

**T1.1 Análise da Query**

Trata-se de uma consulta extremamente demorada, pois envolve as tabelas de Boletos e suas respectivas parcelas dos últimos 5 anos. Possivelmente, ainda existem complicadores como dados que não estão tratados para uso imediato do processamento em questão e que podem demandar algum tipo de tratamento durante a execução deste caso de uso.

**T1.2 Análise do Loop**

Logo após a execução da **Query (T1.1)**, percebe-se que existe um **Loop** o qual irá realizar várias chamadas a uma API externa (**ERP SaaS**) através do **OIC**.

São previstas 1.000 chamadas a esta API o que invariavelmente irá causa um excesso de chamadas à API de origem no **ERP SaaS** com uma sobrecarga e delay para completar o processamento.
Isto ocorre porque o **OIC** é capaz de receber as requisições em um tempo X e faz a requisição ao **ERP SaaS** capaz de atender a um tempo Y. O tempo X é menor que o tempo Y, causando várias chamadas em espera durante um certo tempo.
O risco aqui é a abertura de inúmeras threads em espera para completar o processamento todo, o que pode acarretar mais demora no processamento.

**T1.3 Análise do Processamento OUTROS**

Entende-se por **OUTROS** os demais processamentos para completar o ciclo deste caso de uso que é processar os boletos em atraso dos últimos 5 anos.
Isto pode envolver outras chamadas a APIs externas, OIC entre outros processamentos como Loops e Updates em Banco de Dados.

Não iremos detalhar esta etapa porém trataremos as alternativas cabíveis adiante.

----
# T2 - Possíveis Soluções

**T2.1 Substituir Consulta Única por Lotes**

Dentro do OIC (na figura abaixo, grifado em azul) são executadas inúmeras chamadas (1.000x) para a API do **ERP SaaS** ocasionando o efeito analisado em **T1.2**.


![Fig 8](https://github.com/hoshikawa2/OIC-CASE-1/blob/master/Images/Fig8.jpg?raw=true)

