# Servidores de Parâmetros
:label:`sec_parameterserver`


À medida que mudamos de GPUs únicas para várias GPUs e depois para vários servidores contendo várias GPUs, possivelmente todos espalhadas por vários racks e switches de rede, nossos algoritmos para treinamento distribuído e paralelo precisam se tornar muito mais sofisticados. Os detalhes são importantes, já que diferentes interconexões têm larguras de banda muito diferentes (por exemplo, NVLink pode oferecer até 100 GB/s em 6 links em uma configuração apropriada, PCIe 3.0 16x lanes oferecem 16 GB/s, enquanto mesmo Ethernet de 100 GbE de alta velocidade atinge apenas 10 GB/s) . Ao mesmo tempo, não é razoável esperar que um modelador estatístico seja um especialista em redes e sistemas.

A ideia central do servidor de parâmetros foi introduzida em :cite:`Smola.Narayanamurthy.2010` no contexto de modelos de variáveis ​​latentes distribuídas. Uma descrição da semântica push e pull seguida em :cite:`Ahmed.Aly.Gonzalez.ea.2012` e uma descrição do sistema e uma biblioteca de código aberto seguida em :cite:`Li.Andersen.Park.ea.2014`. A seguir, iremos motivar os componentes necessários para a eficiência.

## Treinamento Paralelo de Dados

Vamos revisar a abordagem de treinamento paralelo de dados para treinamento distribuído. Usaremos isso com a exclusão de todos os outros nesta seção, uma vez que é significativamente mais simples de implementar na prática. Praticamente não há casos de uso (além do aprendizado profundo em gráficos) onde qualquer outra estratégia de paralelismo é preferida, já que as GPUs têm muita memória hoje em dia. :numref:`fig_parameterserver` descreve a variante de paralelismo de dados que implementamos na seção anterior. O aspecto principal nisso é que a agregação de gradientes ocorre na GPU0 antes que os parâmetros atualizados sejam retransmitidos para todas as GPUs.

![Na esquerda: treinamento de GPU único; Na direita: uma variante do treinamento multi-GPU. Ele procede da seguinte maneira. (1) calculamos a perda e o gradiente, (2) todos os gradientes são agregados em uma GPU, (3) a atualização dos parâmetros acontece e os parâmetros são redistribuídos para todas as GPUs.](../img/ps.svg)
:label:`fig_parameterserver`

Em retrospecto, a decisão de agregar na GPU0 parece bastante ad-hoc. Afinal, podemos muito bem agregar na CPU. Na verdade, poderíamos até decidir agregar alguns dos parâmetros em uma GPU e alguns outros em outra. Desde que o algoritmo de otimização suporte isso, não há razão real para que não o possamos fazer. Por exemplo, se tivermos quatro vetores de parâmetro $\mathbf{v}_1, \ldots, \mathbf{v}_4$ com gradientes associados $\mathbf{g}_1, \ldots, \mathbf{g}_4$ poderíamos agreguar os gradientes em uma GPU cada.

$$\mathbf{g}_{i} = \sum_{j \in \mathrm{GPUs}} \mathbf{g}_{ij}$$

Esse raciocínio parece arbitrário e frívolo. Afinal, a matemática é a mesma do começo ao fim. No entanto, estamos lidando com *hardware* físico real onde diferentes barramentos têm diferentes larguras de banda, conforme discutido em :numref:`sec_hardware`. Considere um servidor GPU real de 4 vias conforme descrito em :numref:`fig_bw_hierarchy`. Se estiver bem conectado, pode ter uma placa de rede 100 GbE. Os números mais comuns estão na faixa de 1-10 GbE com uma largura de banda efetiva de 100 MB/s a 1 GB/s. Uma vez que as CPUs têm poucas pistas PCIe para se conectar a todas as GPUs diretamente (por exemplo, CPUs da Intel para consumidores têm 24 pistas), precisamos de um [multiplexador](https://www.broadcom.com/products/pcie-switches-bridges/ chaves pcie). A largura de banda da CPU em um link Gen3 16x é de 16 GB/s. Esta também é a velocidade na qual *cada* uma das GPUs é conectada ao *switch*. Isso significa que é mais eficaz a comunicação entre os dispositivos.

![Um servidor GPU de 4 vias.](../img/bw-hierarchy.svg)
:label:`fig_bw_hierarchy`

Para o efeito do argumento, vamos supor que os gradientes 'pesam' 160 MB. Nesse caso, leva 30ms para enviar os gradientes de todas as 3 GPUs restantes para a quarta (cada transferência leva 10 ms = 160 MB / 16 GB/s). Adicione mais 30ms para transmitir os vetores de peso de volta, chegamos a um total de 60ms.
Se enviarmos todos os dados para a CPU, incorremos em uma penalidade de 40ms, pois *cada* uma das quatro GPUs precisa enviar os dados para a CPU, resultando em um total de 80ms. Por último, suponha que somos capazes de dividir os gradientes em 4 partes de 40 MB cada. Agora podemos agregar cada uma das partes em uma GPU *diferente simultaneamente*, já que o switch PCIe oferece uma operação de largura de banda total entre todos os links. Em vez de 30 ms, isso leva 7,5 ms, resultando em um total de 15 ms para uma operação de sincronização. Resumindo, dependendo de como sincronizamos os parâmetros, a mesma operação pode levar de 15ms a 80ms. :numref:`fig_ps_distributed` descreve as diferentes estratégias para a troca de parâmetros.

![Estratégias de sincronização.](../img/ps-distributed.svg)
:label:`fig_ps_distributed`

Observe que temos mais uma ferramenta à nossa disposição quando se trata de melhorar o desempenho: em uma rede profunda, leva algum tempo para calcular todos os gradientes de cima para baixo. Podemos começar a sincronizar gradientes para alguns grupos de parâmetros, mesmo enquanto ainda estamos ocupados computando-os para outros (os detalhes técnicos disso estão um tanto envolvidos). Veja, por exemplo, :cite:`Sergeev.Del-Balso.2018` para obter detalhes sobre como fazer isso em [Horovod](https://github.com/horovod/horovod).

## Sincronização em Anel

Quando se trata de sincronização em *hardware* de *deep learning* moderno, frequentemente encontramos conectividade de rede significativamente personalizada. Por exemplo, as instâncias AWS P3.16xlarge e NVIDIA DGX-2 compartilham a estrutura de conectividade de :numref:`fig_nvlink`. Cada GPU se conecta a uma CPU host por meio de um link PCIe que opera no máximo a 16 GB/s. Além disso, cada GPU também possui 6 conexões NVLink, cada uma das quais é capaz de transferir 300 Gbit/s bidirecionalmente. Isso equivale a cerca de 18 GB/s por link por direção. Resumindo, a largura de banda NVLink agregada é significativamente maior do que a largura de banda PCIe. A questão é como usá-lo de forma mais eficiente.

![Conectividade NVLink em servidores 8GPU V100 (imagem cortesia da NVIDIA).](../img/nvlink.svg)
:label:`fig_nvlink`

Acontece que :cite:`Wang.Li.Liberty.ea.2018` a estratégia de sincronização ideal é decompor a rede em dois anéis e usá-los para sincronizar os dados diretamente. :numref:`fig_nvlink_twoloop` ilustra que a rede pode ser decomposta em um anel (1-2-3-4-5-6-7-8-1) com largura de banda NVLink dupla e em um (1-4-6-3 -5-8-2-7-1) com largura de banda regular. Projetar um protocolo de sincronização eficiente nesse caso não é trivial.

![Decomposição da rede NVLink em dois anéis.](../img/nvlink-twoloop.svg)
:label:`fig_nvlink_twoloop`

Considere o seguinte experimento de pensamento: dado um anel de $n$  nós de computação (ou GPUs), podemos enviar gradientes do primeiro para o segundo nó. Lá, ele é adicionado ao gradiente local e enviado para o terceiro nó e assim por diante. Após $n-1$ passos, o gradiente agregado pode ser encontrado no último nó visitado. Ou seja, o tempo para agregar gradientes cresce linearmente com o número de nós. Mas, se fizermos isso, o algoritmo será bastante ineficiente. Afinal, a qualquer momento, há apenas um dos nós se comunicando. E se quebrássemos os gradientes em $n$ pedaços e começássemos a sincronizar o pedaço $i$ começando no nó $i$. Como cada pedaço tem o tamanho $1/n$, o tempo total agora é $(n-1)/n \approx 1$ Em outras palavras, o tempo gasto para agregar gradientes *não aumenta* à medida que aumentamos o tamanho do anel. Este é um resultado surpreendente. :numref:`fig_ringsync` ilustra a sequência de etapas em $n=4$ nodes.

![Sincronização de anel em 4 nós. Cada nó começa a transmitir partes de gradientes para seu vizinho esquerdo até que o gradiente montado possa ser encontrado em seu vizinho direito.](../img/ringsync.svg)
:label:`fig_ringsync`

Se usarmos o mesmo exemplo de sincronização de 160 MB em 8 GPUs V100, chegaremos a aproximadamente $2 \cdot 160 \mathrm{MB} / (3 \cdot 18 \mathrm{GB/s}) \approx 6 \mathrm{ms}$. Isto é um pouco melhor do que usar o barramento PCIe, embora agora estejamos usando 8 GPUs. Observe que, na prática, esses números são um pouco piores, uma vez que os *frameworks* de aprendizado profundo geralmente falham em reunir a comunicação em grandes transferências de burst. Além disso, o tempo é crítico.
Observe que há um equívoco comum de que a sincronização em anel é fundamentalmente diferente de outros algoritmos de sincronização. A única diferença é que o caminho de sincronização é um pouco mais elaborado quando comparado a uma árvore simples.

## Treinamento Multi-Máquina

O treinamento distribuído em várias máquinas adiciona mais um desafio: precisamos nos comunicar com servidores que estão conectados apenas por meio de uma malha de largura de banda comparativamente mais baixa, que pode ser mais do que uma ordem de magnitude mais lenta em alguns casos. A sincronização entre dispositivos é complicada. Afinal, diferentes máquinas que executam código de treinamento terão velocidades sutilmente diferentes. Portanto, precisamos *sincronizá-los* se quisermos usar a otimização distribuída síncrona. :numref:`fig_ps_multimachine` ilustra como ocorre o treinamento paralelo distribuído.

1. Um lote (diferente) de dados é lido em cada máquina, dividido em várias GPUs e transferido para a memória da GPU. Essas previsões e gradientes são calculados em cada lote de GPU separadamente.
2. Os gradientes de todas as GPUs locais são agregados em uma GPU (ou, alternativamente, partes dela são agregadas em diferentes GPUs.
3. Os gradientes são enviados para a CPU.
4. A CPU envia os gradientes para um servidor de parâmetros central que agrega todos os gradientes.
5. Os gradientes agregados são então usados para atualizar os vetores de peso e os vetores de peso atualizados são transmitidos de volta para as CPUs individuais.
6. As informações são enviadas para uma (ou várias) GPUs.
7. Os vetores de peso atualizados são espalhados por todas as GPUs.

![Treinamento paralelo distribuído em várias máquinas e em várias GPUs.](../img/ps-multimachine.svg)
:label:`fig_ps_multimachine`

Cada uma dessas operações parece bastante direta. E, de fato, eles podem ser realizadas com eficiência *dentro* de uma única máquina. Quando olhamos para várias máquinas, no entanto, podemos ver que o servidor de parâmetros central se torna o gargalo. Afinal, a largura de banda por servidor é limitada, portanto, para $m$ trabalhadores, o tempo que leva para enviar todos os gradientes ao servidor é $O(m)$. Podemos quebrar essa barreira aumentando o número de servidores para $n$. Neste ponto, cada servidor só precisa armazenar $O(1/n)$ dos parâmetros, portanto, o tempo total para atualizações e otimização torna-se $O(m/n)$. Combinar os dois números resulta em um escalonamento constante, independentemente de quantos trabalhadores estamos lidando. Na prática, usamos as *mesmas* máquinas tanto como trabalhadores quanto como servidores. :numref:`fig_ps_multips` ilustra o design. Veja também :cite:`Li.Andersen.Park.ea.2014` para detalhes. Em particular, garantir que várias máquinas funcionem sem atrasos excessivos não é trivial. Omitimos detalhes sobre barreiras e apenas abordaremos brevemente as atualizações síncronas e assíncronas abaixo.

![Topo - um único servidor de parâmetro é um gargalo, pois sua largura de banda é finita. Parte inferior - servidores de vários parâmetros armazenam partes dos parâmetros com largura de banda agregada.](../img/ps-multips.svg)
:label:`fig_ps_multips`

## Armazenamento de (key,value)

A implementação das etapas necessárias para o treinamento distribuído de várias GPUs na prática não é trivial. Em particular, dadas as muitas opções diferentes que podemos encontrar. É por isso que vale a pena usar uma abstração comum, ou seja, a de um armazenamento (key, value) com semântica de atualização redefinida. Em muitos servidores e muitas GPUs, a computação de gradiente pode ser definida como

$$\mathbf{g}_{i} = \sum_{k \in \mathrm{workers}} \sum_{j \in \mathrm{GPUs}} \mathbf{g}_{ijk}.$$


O aspecto chave nesta operação é que se trata de uma *redução comutativa*, ou seja, ela transforma muitos vetores em um e a ordem de aplicação da operação não importa. Isso é ótimo para nossos propósitos, uma vez que não (precisamos) ter um controle refinado sobre quando qual gradiente é recebido. Observe que é possível realizarmos a redução em etapas. Além disso, observe que esta operação é independente entre os blocos $i$ pertencentes a diferentes parâmetros (e gradientes).

Isso nos permite definir as duas operações a seguir: *push*, que acumula gradientes, e *pull*, que recupera gradientes agregados. Como temos muitos conjuntos diferentes de gradientes (afinal, temos muitas camadas), precisamos indexar os gradientes com a chave $i$. Essa semelhança com armazenamento (key, value), como aquela introduzida no Dynamo
:cite:`DeCandia.Hastorun.Jampani.ea.2007` não é por acaso. Eles também satisfazem muitas características semelhantes, em particular quando se trata de distribuir os parâmetros em vários servidores.


* ***push* (key, value)** envia um gradiente específico (o valor) de um trabalhador para um armazenamento comum. Lá, o parâmetro é agregado, por exemplo, somando-o.
* ***pull* (key, value)** recupera um parâmetro agregado do armazenamento comum, por exemplo, depois de combinar os gradientes de todos os trabalhadores.

Ao ocultar toda a complexidade sobre a sincronização por trás de uma operação simples de *push* e *pull*, podemos dissociar as preocupações do modelador estatístico que deseja ser capaz de expressar a otimização em termos simples e do engenheiro de sistemas que precisa lidar com a complexidade inerente à sincronização distribuída. Na próxima seção, faremos experiências com esse armazenamento (key, value) na prática.

## Resumo

* A sincronização precisa ser altamente adaptável à infraestrutura de rede específica e à conectividade em um servidor. Isso pode fazer uma diferença significativa no tempo que leva para sincronizar.
* A sincronização de anel pode ser ideal para servidores P3 e DGX-2. Para outros, possivelmente nem tanto.
* Uma estratégia de sincronização hierárquica funciona bem ao adicionar vários servidores de parâmetros para aumentar a largura de banda.
* A comunicação assíncrona (enquanto a computação ainda está em andamento) pode melhorar o desempenho.

## Exercícios

1. Você pode aumentar ainda mais a sincronização do toque? Dica: você pode enviar mensagens em ambas as direções.
1. Totalmente assíncrono. Alguns atrasos são permitidos?
1. Tolerância a falhas. Como? E se perdermos um servidor? Isso é um problema?
1. *Checkpoint*
1. Agregação de árvores. Você pode fazer isso mais rápido?
1. Outras reduções (semirregamento comutativo).

[Discussions](https://discuss.d2l.ai/t/366)
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTE0MjAwMTAyNjIsLTk2MzI2ODQzOSwtMT
g1MDI4MTI0NSwtMTg3NTc0OTQ5MF19
-->