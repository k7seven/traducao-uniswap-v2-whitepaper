Uniswap V2 Contratos principais

Autores:
Hayden Adams
hayden@uniswap.org
Noah Zinsmeister
noah@uniswap.org
Dan Robinson
dan@paradigm.xyz

Data:
March 2020

Abstrato

Este whitepaper técnico explica algumas das decisões de design por trás dos Contratos principais da Uniswap v2. Este documento cobre novas características - incluindo pares arbitrários entre ERC20s, um preço de oracle endurecido que permite outros contratos estimarem o preço médio ponderado pelo tempo dentro de um intervalo, flash swaps que permitem traders a receberem ativos e usá-los em outros lugares antes de pagar de volta mais tarde na transação, e uma taxa de protocolo que poderá ser ativada no futuro. Também rearquiteta os contratos para reduzir a superfície de ataque. Este whitepaper descreve os mecanismos dos contratos principais da Uniswap v2, incluindo o contrato do par que armazena os fundos dos provedores de liquidez e o contrato de fábrica usado para iniciar os contratos dos pares.

1. Introducao
