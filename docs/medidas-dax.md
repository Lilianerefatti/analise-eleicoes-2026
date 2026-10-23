# Medidas DAX selecionadas

Esta documentação apresenta três medidas utilizadas no dashboard e as decisões de cálculo envolvidas.

As fórmulas dependem das tabelas e medidas do modelo; não são scripts independentes.

## 1. Diferença de votação em pontos percentuais

```dax
Diferenca Flavio Lula pp =
100 * ([Percentual Flavio] - [Percentual Lula])
```

Calcula a diferença entre os percentuais de votos válidos dos dois candidatos no recorte selecionado.

- Resultado positivo: vantagem de Flávio entre os dois candidatos.
- Resultado negativo: vantagem de Lula entre os dois candidatos.
- Resultado zero: percentuais iguais.

A multiplicação por 100 converte a diferença de proporções em pontos percentuais. Por exemplo, 47% menos 45% resulta em **2 p.p.**

**Formatação:** número decimal com duas casas, sem formato de porcentagem.

## 2. Taxa de abstenção de referência da UF

```dax
Taxa Abstencao Referencia UF =
VAR UFAtual =
    SELECTEDVALUE(Dim_Localidade[SG_UF])
RETURN
    IF(
        NOT ISBLANK(UFAtual),
        CALCULATE(
            [Taxa Abstencao],
            REMOVEFILTERS(Dim_Localidade),
            TREATAS(
                { UFAtual },
                Dim_Localidade[SG_UF]
            )
        ),
        BLANK()
    )
```

Calcula a taxa de abstenção do estado correspondente ao contexto atual.

A medida identifica a UF antes de remover os filtros da dimensão de localidades. Em seguida, reaplica somente essa UF, permitindo comparar a taxa municipal com a taxa estadual completa.

Filtros de outras tabelas permanecem sujeitos ao contexto do modelo.

Quando o contexto não identifica uma única UF, retorna em branco.

**Formatação:** porcentagem com duas casas decimais.

## 3. Correlação entre escolaridade e diferença de votação

```dax
Correlacao Escolaridade Votacao =
VAR Municipios =
    FILTER(
        VALUES(Dim_Localidade[Chave_Localidade]),
        CALCULATE(
            SELECTEDVALUE(Dim_Localidade[SG_UF])
        ) <> "ZZ"
    )
VAR Dados =
    FILTER(
        ADDCOLUMNS(
            Municipios,
            "@X", CALCULATE([Percentual Escolaridade Eleitorado]),
            "@Y", CALCULATE([Diferenca Flavio Lula pp])
        ),
        NOT ISBLANK([@X])
            && NOT ISBLANK([@Y])
    )
VAR N = COUNTROWS(Dados)
VAR MediaX = AVERAGEX(Dados, [@X])
VAR MediaY = AVERAGEX(Dados, [@Y])
VAR Numerador =
    SUMX(
        Dados,
        ([@X] - MediaX) * ([@Y] - MediaY)
    )
VAR SomaQuadradosX =
    SUMX(Dados, POWER([@X] - MediaX, 2))
VAR SomaQuadradosY =
    SUMX(Dados, POWER([@Y] - MediaY, 2))
RETURN
    IF(
        HASONEVALUE(
            Fato_Perfil_Eleitorado[DS_GRAU_ESCOLARIDADE]
        ) && N >= 3,
        DIVIDE(
            Numerador,
            SQRT(SomaQuadradosX * SomaQuadradosY)
        ),
        BLANK()
    )
```

Calcula a correlação de Pearson entre:

- **X:** participação da escolaridade selecionada no eleitorado municipal.
- **Y:** diferença de votação Flávio − Lula, em pontos percentuais.

### Regras do cálculo

- Exclui os registros do exterior, identificados por `ZZ`.
- Utiliza apenas municípios com valores não vazios nas duas variáveis.
- Preserva valores iguais a zero como observações válidas.
- Exige uma única categoria de escolaridade no contexto.
- Exige pelo menos três municípios com pares válidos.
- Atribui o mesmo peso a cada município.
- Retorna em branco quando não há variação em uma das variáveis, pois o denominador é zero.

O resultado agregado é recalculado com os municípios do recorte selecionado. Não corresponde à média das correlações regionais.

### Interpretação

O coeficiente varia de −1 a +1. O sinal indica a direção da associação, e a proximidade dos extremos indica maior intensidade da associação linear.

Correlação não demonstra causalidade e não permite identificar como pessoas de determinada escolaridade votaram.

**Formatação:** número decimal com três casas, sem formato de porcentagem.
