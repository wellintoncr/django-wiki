# Django wiki

## Setup
```
pip install -r requirements.txt
```

## Desenvolvimento
```
mkdocs serve -a localhost:9000
```
Ao salvar algum arquivo, o servidor será reiniciado e exibirá o conteúdo atualizado.

## Deploy
```
mkdocs gh-deploy
```
O deploy pegará o que está na branch atual e mandar para a branch `gh-pages`, incluindo arquivos que ainda não foram versionados. Antes de executar o comando, verifique se a documentação realmente poderá ser publicada.

Esse processo já fará `push` e publicará automaticamente.