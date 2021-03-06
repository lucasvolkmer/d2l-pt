# Visão geral dos sistemas de recomendação


Na última década, a Internet evoluiu para uma plataforma para serviços online de grande escala, o que mudou profundamente a maneira como nos comunicamos, lemos notícias, compramos produtos e assistimos filmes. Nesse ínterim, o número sem precedentes de itens (usamos o termo *item* para nos referir a filmes, notícias, livros e produtos) oferecidos online requer um sistema que pode nos ajudar a descobrir os itens de nossa preferência. Os sistemas de recomendação são, portanto, ferramentas poderosas de filtragem de informações que podem facilitar serviços personalizados e fornecer experiência sob medida para usuários individuais. Em suma, os sistemas de recomendação desempenham um papel fundamental na utilização da riqueza de dados disponíveis para fazer escolhas gerenciáveis. Hoje em dia, os sistemas de recomendação estão no centro de vários provedores de serviços online, como Amazon, Netflix e YouTube. Lembre-se do exemplo de livros de aprendizagem profunda recomendados pela Amazon em :numref:`subsec_recommender_systems`. Os benefícios de empregar sistemas de recomendação são duplos: por um lado, pode reduzir muito o esforço dos usuários em encontrar itens e aliviar a questão da sobrecarga de informações. Por outro lado, pode agregar valor ao negócio online
prestadores de serviços e é uma importante fonte de receita. Este capítulo irá apresentar os conceitos fundamentais, modelos clássicos e avanços recentes com aprendizado profundo no campo de sistemas de recomendação, juntamente com exemplos implementados.

![Illustration of the Recommendation Process](../img/rec-intro.svg)


## Filtragem colaborativa

Começamos a jornada com o conceito importante em sistemas de recomendação ---filtragem colaborativa
(CF), que foi cunhado pela primeira vez pelo sistema Tapestry :cite:`Goldberg.Nichols.Oki.ea.1992`, referindo-se a "as pessoas colaboram para se ajudarem a realizar o processo de filtragem a fim de lidar com as grandes quantidades de e-mail e mensagens postadas em grupos de notícias". Este termo foi enriquecido com mais sentidos. Em um sentido amplo, é o processo de
filtragem de informações ou padrões usando técnicas que envolvem colaboração entre vários usuários, agentes e fontes de dados. FC tem muitas formas e vários métodos de FC propostos desde seu advento.

No geral, as técnicas de CF podem ser categorizadas em: CF baseado em memória, CF baseado em modelo e seus híbridos :cite:`Su.Khoshgoftaar.2009`. As técnicas representativas de CF baseado em memória são o CF baseado no vizinho mais próximo, como CF baseado em usuário e CF baseado em item :cite:`Sarwar.Karypis.Konstan.ea.2001`. Modelos de fator latente, como fatoração de matriz, são exemplos de CF baseado em modelo. O CF baseado em memória tem limitações para lidar com dados esparsos e em grande escala, uma vez que calcula os valores de similaridade com base em itens comuns. Métodos baseados em modelos tornam-se mais populares com seus
melhor capacidade para lidar com esparsidade e escalabilidade. Muitas abordagens de CF baseadas em modelo podem ser estendidas com redes neurais, levando a modelos mais flexíveis e escaláveis ​​com a aceleração de computação no aprendizado profundo :cite:`Zhang.Yao.Sun.ea.2019`. Em geral, o CF usa apenas os dados de interação do item do usuário para fazer previsões e recomendações. Além do CF, os sistemas de recomendação baseados em conteúdo e contexto também são úteis para incorporar as descrições de conteúdo de itens/usuários e sinais contextuais, como carimbos de data/hora e locais. Obviamente, podemos precisar ajustar os tipos/estruturas do modelo quando diferentes dados de entrada estiverem disponíveis.



## Feedback explícito e feedback implícito

Para saber a preferência dos usuários, o sistema deve coletar feedback deles. O feedback pode ser explícito ou implícito :cite:`Hu.Koren.Volinsky.2008`. Por exemplo, [IMDB](https://www.imdb.com/) coleta classificações por estrelas que variam de uma a dez estrelas para filmes. O YouTube fornece os botões de polegar para cima e para baixo para que os usuários mostrem suas preferências. É evidente que a coleta de feedback explícito exige que os usuários indiquem seus interesses de forma proativa. No entanto, o feedback explícito nem sempre está disponível, pois muitos usuários podem relutar em avaliar os produtos. Relativamente falando, o feedback implícito está frequentemente disponível, uma vez que se preocupa principalmente com a modelagem do comportamento implícito, como cliques do usuário. Como tal, muitos sistemas de recomendação são centrados no feedback implícito que reflete indiretamente a opinião do usuário por meio da observação do comportamento do usuário. Existem diversas formas de feedback implícito, incluindo histórico de compras, histórico de navegação, relógios e até movimentos do mouse. Por exemplo, um usuário que comprou muitos livros do mesmo autor provavelmente gosta desse autor. Observe que o feedback implícito é inerentemente barulhento. Nós podemos apenas *adivinhar* suas preferências e verdadeiros motivos. Um usuário assistiu a um filme não indica necessariamente uma visão positiva desse filme.



## Tarefas de recomendação

Várias tarefas de recomendação foram investigadas nas últimas décadas. Com base no domínio de aplicativos, há recomendação de filmes, recomendações de notícias, recomendação de ponto de interesse :cite:`Ye.Yin.Lee.ea.2011` e assim por diante. Também é possível diferenciar as tarefas com base nos tipos de feedback e dados de entrada, por exemplo, a tarefa de previsão de classificação visa prever as classificações explícitas. A recomendação Top-$n$ (classificação de itens) classifica todos os itens de cada usuário pessoalmente com base no feedback implícito. Se as informações de carimbo de data / hora também forem incluídas, podemos construir uma recomendação ciente de sequência :cite:`Quadrana.Cremonesi.Jannach.2018`. Outra tarefa popular é chamada de previsão de taxa de cliques, que também é baseada em feedback implícito, mas vários recursos categóricos podem ser utilizados. Recomendar para novos usuários e recomendar novos itens para usuários existentes são chamados de recomendação de inicialização a frio :cite:`Schein.Popescul.Ungar.ea.2002`.



## Sumário

* Os sistemas de recomendação são importantes para usuários individuais e indústrias. A filtragem colaborativa é um conceito-chave na recomendação.
* Existem dois tipos de feedbacks: feedback implícito e feedback explícito. Várias tarefas de recomendação foram exploradas durante a última década.

## Exercícios

1. Você pode explicar como os sistemas de recomendação influenciam sua vida diária?
2. Que tarefas de recomendação interessantes você acha que podem ser investigadas?

[Discussão](https://discuss.d2l.ai/t/398)
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTk4MzAzNTI5OV19
-->