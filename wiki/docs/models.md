# Models

O sistema de modelagem do Django é bem completo e possui diversas funções úteis. Abaixo estão listadas algumas funcionalidades interessantes.

Como alguns trechos de código podem depender de qual banco de dados está sendo usado, este documento considera o Postgres.
Caso esteja usando outro tipo, verifique a compatibilidade dos comandos.

## View

Pode ser que você queira usar uma view na sua aplicação. Você pode realizar esse processo de duas formas:

1. Manual: você precisa acessar o banco de dados, executar o SQL que cria a view e, na aplicação Django, executar um raw SQL quando precisar de algum dado da view;

1. Integrado: você criará uma migração customizada e adicionará um modelo na aplicação. Quando precisar dos dados da view, basta acessar o modelo.

A forma manual é complicada porque necessita que você entre em cada banco de dados e execute o mesmo SQL (imagine fazendo isso em ambiente local, homologação e produção), além de necessitar executar um raw SQL na aplicação Django - dificultando bastante o uso de recursos nativos, como uma ListView que pagina os resultados automaticamente.

Dessa forma, o foco é realizar esta operação de forma integrada.

### Forma integrada

Comece criando um arquivo de migração. Na pasta `migrations`, crie um arquivo que vem após a última migração.

Para exemplificar, considere que a última migração foi `0010_create_model.py`.
A migração que você deve criar é `0011_create_view.py` (o que vem após `0011_` é irrelevante).

Os dois exemplos de implementação se diferenciam no tipo de view criada.

Uma view comum (ou virtual) é simplesmente um jeito de rodar uma query de outra forma, exemplo:
```
SELECT id, price FROM product;
```

E uma view pode ser simplesmente acessar os produtos que estão ativos:
```
CREATE VIEW my_view AS SELECT id, price FROM product WHERE status = 'active';
```

Então, ao fazer um *SELECT* de `my_view`, apenas os produtos ativos estarão presentes enquanto que a tabela `product` conterá todos os produtos (ativos ou não).
Caso algum produto seja alterado, `my_view` será automaticamente atualizada - por baixo dos panos, a query original será executada, então os registros sempre estarão atualizados.

Porém, pode ser que você tenha muitos dados a serem processados e é necessário realizar algum tipo de cache. Neste caso, uma view materializada é uma opção.

Quando a view é criada, ela é executada e os resultados ficam armazenados no banco de dados. Caso algum registro mude, a view não é atualizada automaticamente.
Para que os dados da view sejam sincronizados com os registros da query original, é necessário executar um comando específico.

Independente de qual tipo de view será usada, você precisa especificar o parâmetro *sql* de *RunSQL* e, opcionalmente, *reverse_sql* pode ser especificado para ser usado caso seja necessário reverter a migração.

#### View comum

Neste arquivo novo, crie um SQL que vai gerar a view:

``` py title="0011_create_view.py"
from django.db import migrations


class Migration(migrations.Migration):
    dependencies = [
        ("core", "0010_create_model"),
    ]

    operations = [
        migrations.RunSQL(
            sql="""
            CREATE VIEW app_view_name AS
            (
                SELECT o.price, o.id
                FROM app_order o
            );
            """,
            reverse_sql="""
            DROP VIEW IF EXISTS app_view_name;
            """,
        ),
    ]

```

#### View materializada

Neste arquivo novo, crie um SQL que vai gerar a view:

``` py title="0011_create_view.py"
from django.db import migrations


class Migration(migrations.Migration):
    dependencies = [
        ("core", "0010_create_model"),
    ]

    operations = [
        migrations.RunSQL(
            sql="""
            CREATE MATERIALIZED VIEW app_view_name AS
            (
                SELECT o.price, o.id
                FROM app_order o
            );
            CREATE UNIQUE INDEX app_order_id ON app_view_name (id);
            """,
            reverse_sql="""
            DROP MATERIALIZED VIEW IF EXISTS app_view_name;
            """,
        ),
    ]

```

Neste caso, você também precisará de alguma rotina que fará o sincronismo dos dados. Isso pode ser feito através de um script localizado na pasta `app/management/commands` (sendo `app` o nome do app).
Segue um exemplo:

```py title="sync_materialized_view.py"
from django.core.management.base import BaseCommand
from django.db import connection


class Command(BaseCommand):
    def handle(self, *args, **options):
        with connection.cursor() as cursor:
            cursor.execute(f"REFRESH MATERIALIZED VIEW CONCURRENTLY app_view_name;")

```

No comando acima, atenção ao `CONCURRENTLY` porque indica que o banco de dados não travará *SELECT*s concorrentes, permitindo que ações de *SELECT* possam ocorrer durante a sincronização.
Porém, para que isso seja possível, é necessário que haja pelo menos um índice na view (o exemplo do arquivo de migração contém o trecho que cria este índice).

Caso não seja possível criar o índice, não coloque `CONCURRENTLY` e tenha ciência que, durante a sincronização, o view estará inacessível.

Obviamente, existe um preço a ser pago: adicionar o modo concorrente pode aumentar o tempo de execução, então avalie o seu caso antes de escolher qual opção usar.

### Modelo

Uma vez efetuada a criação da migração, modifique o arquivo de modelos e adicione a nova view:
```py title="models.py"
class MyView(models.Model):
    id = models.BigIntegerField(primary_key=True)
    price = models.DecimalField(max_digits=5, decimal_places=2)

    class Meta:
        db_table = "app_view_name"
        managed = False

```
Do exemplo acima, o mais importante é definir corretamente o `db_table` para ter o mesmo nome da view e `managed = False` para que o Django não tente criar as tabelas e somente vincular ao `db_table` informado. Feito isso, execute `python manage.py makemigrations`.