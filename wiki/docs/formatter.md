# Formatadores

Formatadores úteis

## Dinheiro (ou similares)
No template, coloque:
```
variavel|floatformat:"2g"
```
A formatação vai considerar a localização. Supondo que `variavel` seja 10.33333333, então, se o locale for português, o resultado ficará como `10,33`.

Outros exemplos:

| Valor        | Saída       |
| ------------ | ----------- |
| `10.366666`  | `10,37`     |
| `1234.366666`| `1.124,37`  |
| `10.0`       | `10,00`     |

O argumento `2` indica a quantidade de casas decimais (sempre com duas casas decimais, no caso). O `g` indica que haverá agrupamento de milhar de acordo com o locale (em inglês americano, por exemplo, o separador é `,`).

[A documentação completa pode ser acessada aqui](https://docs.djangoproject.com/en/dev/ref/templates/builtins/#floatformat).

## Forma humanizada
Às vezes pode ser necessário aplicar uma forma humanizada a algum valor. Para tal, o seguinte setup é necessário:

Primeiro, verifique se, em `settings.py`, o app `humanize` está incluso:
```
INSTALLED_APPS = [
    ...,
    "django.contrib.humanize",
]
```
No template que ocorrerá a conversão, carregue o `humanize`:
```
{% load humanize %}
(restante do HTML)
```
Todas as formas humanizadas abaixo necessitam deste setup.

[Documentação](https://docs.djangoproject.com/en/5.1/ref/contrib/humanize/#module-django.contrib.humanize)

## Forma humanizada de números grandes (milhão, trilhão, etc)
[Setup](#forma-humanizada)

Na variável a ser transformada, aplique um filtro:

```
variavel|intword
```

Alguns exemplos considerando o locale como português brasileiro:

| Valor      | Saída         |
| ---------- | -----------   |
| `1_000`    | `1000`        |
| `1_000_000`| `1,0 milhão`  |
| `1_200_000`| `1,2 milhões` |

## Forma humanizada de datas (ontem, hoje e amanhã)
[Setup](#forma-humanizada)

Na variável a ser transformada, aplique um filtro:

```
variavel|naturalday
```

Alguns exemplos considerando o locale como português brasileiro e que hoje é 02/01/2000:

| Valor                | Saída                   |
| -------------------- | ----------------------- |
| `date(2000, 1, 2)`   | `hoje`                  |
| `date(2000, 1, 1)`   | `ontem`                 |
| `date(2000, 1, 3)`   | `amanhã`                |
| `date(2000, 1, 4)`   | `04 de Janeiro de 2000` |
