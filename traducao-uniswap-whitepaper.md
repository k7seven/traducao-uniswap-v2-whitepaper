Essa tradução foi escrita por k77 manualmente, alguns dos termos foram mantidos em inglês para melhor entendimento, pois já fazem parte da linguagem da comunidade DeFi.
Não foi usado nenhum tipo de software para tradução do texto. Esta tradução foi realizada apenas com o objetivo de facilitar o aprendizado e o entendimento geral sobre o funcionamento do Uniswap V2.

The translation below was written by k77 manually, some of the terms were kept in english for better understanding, because the terms are already part of the language of the DeFi community.
No translation software tools were used in the translation. The translation was written only with learning purpose to understand better in a general way how the Uniswap v2 works.

Uniswap V2 Contratos Principais

Autores:
Hayden Adams
hayden@uniswap.org
Noah Zinsmeister
noah@uniswap.org
Dan Robinson
dan@paradigm.xyz

Data:
Março 2020

Resumo

Este whitepaper técnico explica algumas das decisões de design por trás dos Contratos principais da Uniswap V2. Este documento cobre novas características - incluindo pares arbitrários entre ERC20s, um oráculo de preço fortalecido que permite outros contratos estimarem o preço médio ponderado pelo tempo dentro de um intervalo, flash swaps que permitem traders a receberem ativos e usá-los em outros lugares antes de pagar de volta mais tarde na transação, e uma taxa de protocolo que poderá ser ativada no futuro. Também rearquiteta os contratos para reduzir a superfície de ataque. Este whitepaper descreve os mecanismos dos contratos principais da Uniswap V2, incluindo o contrato do par que armazena os fundos dos provedores de liquidez e o contrato de fábrica usado para iniciar os contratos dos pares.

1. Introdução

Uniswap V1 é um sistema on-chain de contratos inteligentes na blockchain da Ethereum, implementando um protocolo de liquidez automatizada baseado em uma fórmula chamada produto constante [1]. Cada par na Uniswap V1 armazena reservas conjuntas de dois ativos, e provê liquidez para aqueles dois ativos, mantendo a invariante de que o produto das reservas não pode diminuir. Traders pagam uma taxa de 30 pontos base em cada trade, que vai para os provedores de liquidez. Os contratos não são atualizáveis.

Uniswap V2 é uma nova implementação baseada na mesma fórmula, com várias novas características altamente desejáveis. A mais significativa, permite a criação de pares arbitrários de ERC20/ERC20, ao invés de suportar pares apenas de ETH/ERC20. Também provê um oráculo de preço fortalecido que acumula o preço relativo de dois ativos no começo de cada bloco. Isso permite outros contratos da Ethereum estimarem o preço médio ponderado pelo tempo para os dois ativos por meio de intervalos arbitrários. Finalmente, permite flash swaps, onde usuários podem receber ativos livremente e usá-los em outros lugares na rede, apenas pagando (ou retornando) pelos ativos no final da transação.

Enquanto o contrato não é no geral atualizável, existe uma chave privada que tem a habilidade de atualizar uma variável no contrato de fábrica para ligar uma taxa de 5 pontos base nos trades. Essa taxa será inicialmente desligada, mas poderá ser ligada no futuro, depois disso os provedores de liquidez irão receber 25 pontos base de cada trade, ao invés de 30 pontos base.

Como discutido na seção 3, Uniswap V2 também corrige alguns pequenos problemas com Uniswap V1, e também rearquiteta a implementação, reduzindo a superfície de ataque da Uniswap e fazendo o sistema mais fácil de ser atualizado ao minimizar a lógica no contrato principal que segura os fundos dos provedores de liquidez.

Este papel descreve as mecânicas dos contratos principais, assim como o contrato de fábrica usado para instanciar os contratos principais. Na verdade, usando Uniswap V2 vai requerer chamada do contrato dos pares através de um contrato roteador que computa o trade ou montante de depósito e transfere fundos para o contrato dos pares.

2. Novas funcionalidades

2.1 Pares ERC20
Uniswap v1 utilizava ETH como uma moeda ponte. Todo par incluía ETH como um dos seus ativos. Isso tornava o roteamento mais simples - todo trade entre ABC e XYZ passava através do par ETH/ABC e ETH/XYZ - e reduzia a fragmentação da liquidez.

Contudo, essa regra impunha custos significativos nos provedores de liquidez. Todos os provedores de liquidez tinham exposição ao ETH e sofriam impermanent loss baseado nas mudanças dos preços de outros ativos em relação ao ETH. Quando dois ativos ABC e XYZ são correlacionados - por exemplo, se os dois são stablecoins de dólar - os provedores de liquidez em um par ABC/XYZ na Uniswap seriam geralmente sujeitos a menos impermanent loss do que os pares ABC/ETH e XYZ/ETH.

Usar ETH como uma moeda ponte mandatória também impunha custos para os traders. Os traders tinham que pagar o dobro em taxas do que eles iriam pagar em pares diretos como ABC/XYZ, e eles sofriam slippage em dobro.

Uniswap v2 permite aos provedores de liquidez criarem contratos de pares para qualquer ERC20.

Uma proliferação de pares entre ERC20 arbitrários poderia tornar um tanto quanto difícil para encontrar o melhor caminho para negociar um par específico, mas o roteamento poderia ser manipulado em uma camada mais alta (tanto off-chain quanto on-chain através de um roteador ou agregador).

2.2 Oráculo de Preço

O preço marginal oferecido pela Uniswap (sem incluir taxas) no tempo t pode ser computado ao dividir as reservas do ativo a pelas reservas do ativo b.

![image1](https://github.com/k7seven/traducao-uniswap-v2-whitepaper/assets/132465200/568f5f09-6501-4ec0-ae20-e5053bc90ef9)

Como os arbitradores vão fazer trade na Uniswap se esse preço estiver incorreto (por um montante suficiente que valha a pena, contando as taxas), o preço oferecido pela Uniswap tende a acompanhar o preço relativo de mercado dos ativos, conforme demonstrado por Angeris e outros. Isso significa que pode ser utilizado como um oráculo de preço aproximado.

Contudo, a Uniswap v1 não é segura para ser utilizada como um oráculo de preço on-chain, porque é fácil de ser manipulado. Suponhamos que outro contrato está usando o preço do par ETH/DAI para liquidar um derivativo. Um atacante que gostaria de manipular o preço medido poderia comprar ETH do par ETH/DAI, que ativaria a liquidação no contrato derivativo (causando a liquidação baseada no preço inflado), e então venderia o ETH de volta para o par para trocar de volta ao verdadeiro preço. Isso poderia ser feito como uma transação atômica, ou por um minerador que controla a ordem das transações dentro de um bloco.

Uniswap v2 melhora essa funcionalidade do oráculo ao medir e gravar o preço antes do primeiro trade de cada bloco (ou equivalente, depois do último trade do bloco anterior). Esse preço é mais difícil de manipular do que preços durante um bloco. Se o atacante enviar uma transação que tenta manipular o preço no final do bloco, algum outro arbitrador poderá enviar outra transação para fazer o trade de volta imediatamente depois no mesmo bloco. Um minerador (ou um atacante que usar gás suficiente para preencher um bloco inteiro) poderia manipular o preço no final do bloco, mas a não ser que eles minerem o próximo bloco também, eles não terão uma vantagem em arbitrar o trade de volta.

Especificamente, a Uniswap v2 acumula o preço, ao manter o acompanhamento da soma cumulativa dos preços no começo de cada bloco em que alguém interage com o contrato. Cada preço é ponderado pela quantidade de tempo que passou desde o último bloco em que foi atualizado, de acordo com o timestamp do bloco. Isso significa que o valor acumulado de cada tempo (depois de ser atualizado) deverá ser a soma do preço spot em cada segundo no histórico do contrato.

![image2](https://github.com/k7seven/traducao-uniswap-v2-whitepaper/assets/132465200/e8d0c1ca-208c-4627-8769-b2d8d08a484a)

Para estimar o preço médio ponderado no tempo do tempo t1 para o t2, um chamador externo pode fazer um checkpoint no valor do acumulador no ponto t1 e depois também no ponto t2, subtrair o primeiro valor do segundo e dividir pelo número de segundos que se passou. (Note que o contrato não armazena histórico de valores desse acumulador - o chamador tem que chamar o contrato no começo do período para ler e armazenar o valor).

![image3](https://github.com/k7seven/traducao-uniswap-v2-whitepaper/assets/132465200/9b9f270f-29e4-46f7-89d6-e8adc8668ecb)

Usuários do oráculo podem escolher quando começar e finalizar esse período. Escolhendo um período mais longo faz com que seja mais caro para um atacante manipular o TWAP, porém faz com que o preço seja menos atualizado.

Uma complicação: deveríamos medir o preço do ativo A em termos do ativo B, ou o preço do ativo B em termos do ativo A? Enquanto o preço spot do ativo A em termos do ativo B sempre é recíproco do preço spot de B em termos de A, o preço médio do ativo A em termos do ativo B dentro de um particular período de tempo não é igual ao recíproco do preço médio do ativo B em termos do ativo A. Por exemplo, se o preço USD/ETH é 100 no bloco 1 e 300 no bloco 2, a média de USD/ETH é 200 USD/ETH, mas a média do par ETH/USD será 1/150 ETH/USD. Como o contrato não pode saber qual dos dois ativos os usuários gostariam de usar como unidade de conta, Uniswap v2 mantém o acompanhamento dos dois preços.

Outra complicação é que é possível para alguém enviar ativos para o contrato do par - conseguindo modificar o balanço e o preço marginal - sem interagir com o contrato, sendo assim não ativando a atualização do oráculo. Se o contrato apenas checou seu próprio balanço e atualizou o oráculo baseado no preço atual, um atacante poderia manipular o oráculo ao enviar um ativo para o contrato imediatamente antes de chamar o contrato pela primeira vez dentro de um bloco. Se o último trade foi dentro de um bloco em que o timestamp era x segundos atrás, o contrato multiplicaria incorretamente o novo preço por x antes de acumulá-lo, mesmo que ninguém tenha uma oportunidade de fazer um trade naquele preço. Para prevenir isso, o contrato principal guarda suas reservas depois de cada interação, e atualiza o oráculo utilizando o preço derivado das reservas guardadas do que as reservas atuais. Além de proteger o oráculo de manipulação, essa mudança permite a re-arquitetura do contrato descrita abaixo na seção 3.2.

2.2.1 Precisão
Como o Solidity não tem suporte de primeira classe para tipos de dados numéricos não-inteiros, o Uniswap v2 usa um formato binário simples de ponto fixo para codificar e manipular preços. Especificamente, preços em um dado momento são armazenados como números UQ112.112, isso quer dizer que 112 bits fracionados de precisão são especificados em algum dos lados dos pontos decimais, sem sinal. Esses números têm uma variação entre [0, 2^112 - 1]^4 e uma precisão de 1/2^112.

O formato UQ112.112 foi escolhido por uma razão pragmática - porque esses números podem ser armazenados como uint224, isso deixa livre 32 bits de 256 bits em um slot de armazenamento. Também acontece que essas reservas, cada armazenamento em um uint112 também deixa 32 bits livres em um slot de 256 bits. Esses espaços livres são utilizados para o processo de acumulação descrito acima. Especificamente, as reservas são armazenadas junto com o timestamp do bloco mais recente com pelo menos 1 trade, modificado com 2^32 para que caiba em 32 bits. Adicionalmente, apesar do preço em qualquer dado momento (armazenado como um número UQ112.112) ser garantido a caber em 224 bits, a acumulação desse preço dentro de um intervalo não é. Os 32 bits extras no final do slot de armazenamento para o preço acumulado de A/B e B/A são utilizados para armazenar bits overflow que vêm de repetidas somas dos preços. Esse design significa que o oráculo de preço adiciona apenas 3 operações SSTORE (custo de 15,000 gas) para o primeiro trade em cada bloco.

O primeiro lado ruim é que 32 bits não é suficiente para armazenar os valores de timestamp que nunca irão overflow. Na realidade, a data quando o Unix timestamp overflows um uint32 é 02/07/2106. Para ter certeza que esse sistema continua a funcionar corretamente depois dessa data, e a cada múltiplo de 2^32 - 1 segundos depois, oráculos simplesmente são requeridos a verificarem os preços pelo menos uma vez por intervalo (aproximadamente 136 anos). Isso é porque o principal método de acumulação (e modificação do timestamp), na realidade é overflow-safe, significando que trades entre intervalos overflow podem ser apropriadamente contabilizados levando em consideração que os oráculos estão usando a correta (mais simples) aritmética de overflow para computar os deltas.
