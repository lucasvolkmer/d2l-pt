# Apêndice: Matemática para *Deep Learning*
:label:`chap_appendix_math`

**Brent Werness** (*Amazon*), **Rachel Hu** (*Amazon*), e autores deste livro



Uma das partes maravilhosas do *deep learning* moderno é o fato de que muito dele pode ser compreendido e usado sem uma compreensão completa da matemática abaixo dele. Isso é um sinal de que o campo está amadurecendo. Assim como a maioria dos desenvolvedores de *software* não precisa mais se preocupar com a teoria das funções computáveis, os profissionais de aprendizado profundo também não devem se preocupar com os fundamentos teóricos do aprendizado de máxima *likelihood*.

Mas ainda não chegamos lá.

Na prática, às vezes você precisará entender como as escolhas arquitetônicas influenciam o fluxo de gradiente ou as suposições implícitas que você faz ao treinar com uma determinada função de perda. Você pode precisar saber o que mede a entropia mundial e como isso pode ajudá-lo a entender exatamente o que cada bit por caractere significam em seu modelo. Tudo isso requer um conhecimento matemático mais profundo.


Este apêndice tem como objetivo fornecer a base matemática que você precisa para entender a teoria central do *deep learning*, moderno, mas não é exaustivo. Começaremos examinando a álgebra linear em maior profundidade. Desenvolvemos uma compreensão geométrica de todos os objetos algébricos lineares comuns e operações que nos permitirão visualizar os efeitos de várias transformações em nossos dados. Um elemento chave é o desenvolvimento das noções básicas de autodescomposições.

Em seguida, desenvolvemos a teoria do cálculo diferencial até o ponto em que podemos entender completamente por que o gradiente é a direção de descida mais acentuada e por que a retropropagação assume a forma que assume. O cálculo integral é então discutido no grau necessário para apoiar nosso próximo tópico, a teoria da probabilidade.

Os problemas encontrados na prática frequentemente não são certos e, portanto, precisamos de uma linguagem para falar sobre coisas incertas. Revisamos a teoria das variáveis ​​aleatórias e as distribuições mais comumente encontradas para que possamos discutir os modelos de forma probabilística. Isso fornece a base para o classificador *naive* Bayes, uma técnica de classificação probabilística.


Intimamente relacionado à teoria da probabilidade está o estudo da estatística. Embora as estatísticas sejam um campo muito grande para fazer justiça em uma seção curta, apresentaremos conceitos fundamentais que todos os praticantes de aprendizado de máquina devem estar cientes, em particular: avaliação e comparação de estimadores, realização de testes de hipótese e construção de intervalos de confiança.

Por último, voltamos ao tópico da teoria da informação, que é o estudo matemático do armazenamento e transmissão da informação. Isso fornece a linguagem central pela qual podemos discutir quantitativamente quanta informação um modelo contém em um domínio do discurso.

Juntos, eles formam o núcleo dos conceitos matemáticos necessários para iniciar o caminho em direção a uma compreensão profunda do *deep learning*.

```toc
:maxdepth: 2

geometry-linear-algebraic-ops
eigendecomposition
single-variable-calculus
multivariable-calculus
integral-calculus
random-variables
maximum-likelihood
distributions
naive-bayes
statistics
information-theory
```

<!--stackedit_data:
eyJoaXN0b3J5IjpbMTA0MTMzNTY0MCwtMTkwMjU5ODY5Ml19
-->