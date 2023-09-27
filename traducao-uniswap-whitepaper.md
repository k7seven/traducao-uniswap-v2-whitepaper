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
