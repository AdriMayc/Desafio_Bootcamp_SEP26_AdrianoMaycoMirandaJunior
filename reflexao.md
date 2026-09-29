## Por que o assistente precisa citar a FONTE e o que aconteceria em uma empresa real se ele respondesse sem citar?

A FONTE é crucial para manter a identificação e a veracidade da informação que chega ao colaborador. Em uma empresa real, responder sem indicar a fonte poderia resultar em decisões tomadas com base em dados incorretos ou desatualizados, o que poderia gerar complicações internas e tornaria mais difícil validar a informação que foi utilizada.

## Qual foi a decisão mais difícil entre chunking, pré-processamento e threshold, e como você a tomou?

A parte mais desafiadora foi escolher o threshold, uma vez que ele varia de acordo com os dados e a representação textual utilizada. Basei-me apenas no texto das seções, analisei os scores e defini o limite em 0,30. Ao experimentar a adição do título e do nome da seção, os scores se alteraram, mostrando que diferentes representações podem precisar de uma nova calibração do threshold.

## Se você tivesse acesso a um LLM, o que mudaria na Parte 4 e que novo risco apareceria?

Com um LLM à disposição, daria para criar respostas em linguagem natural com base nos chunks recuperados, em vez de devolver apenas o texto do chunk.  Porém, isso poderia aumentar o risco de alucinações, já que o modelo poderia criar informações que não estão nas políticas.  Isso destaca o papel crucial da FONTE em possibilitar o rastreamento e a validação da informação.