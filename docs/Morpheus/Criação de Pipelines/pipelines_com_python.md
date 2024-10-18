# Pipelines com Python

Os estágios do Morpheus são conectados iguais a um grafo, onde cada nó é um estágio que recebe os dados e retornam outro, ^^é preciso deixar claro os tipos dos dados de entrada e de saída^^ para que o Morpheus possa conferir na hora da execução da pipeline.

> É possível criar uma função ou uma classe para o estágio, permitindo mais flexibilidade na criação do estágio.

---

## Funções como Estágio

Toda função precisa do decorador `@stage` que é responsavel por deixar claro para o Morpheus que se trata de uma função que será utilizada como um estágio a ser executado.

```py
@stage
def pass_thru_stage(message: typing.Any) -> typing.Any:
    # Retorna mensagem para o proximo estágio (aqui ficaria a lógica do estágio)
    return message

# --- Implementando no Pipeline ---
config = Config()
pipeline = LinearPipeline(config)
# ...
pipeline.add_stage(pass_thru_stage(config))
```

É preciso também deixar claro o tipo da mensagem que será recebida `message: typing.Any` e o tipo do retorno `-> typing.Any` para que o Morpheus possa conferir ao criar o nó.

---

## Classes como Estágio

Ao optarmos por utilizar uma classe, precisamos especificar o tipo da classe herdando alguma pré-definida pelo Morpheus, alguns exemplos:

| Tipo | Descrição |
|------|-----------|
| SinglePortStage         | Estágios que contém apenas uma entrada e saída. |
| SingleOutputSource | Estágios que atuam como fontes de dados, no sentido que não recebe uma entrada de um estágio anterior, por exemplo: algum arquivo ou mensageria com Kafka. |
| PassThruTypeMixin | Define que o tipo da mensagem de entrada é o mesmo do de saída (muito comum no Morpheus). |

!!! note
    Caso queira utilizar a classe especificada no CLI, precisamos utilizar um decorador na classe chamado de ``@register_stage("pass-thru")`` fazendo com que seja possível a detecção na hora da criação da pipeline.
    ```py
    @register_stage("pass-thru")
    class PassThruStage(PassThruTypeMixin, SinglePortStage):
    ```


É preciso criar 5 métodos na subclasse para implementar a interface stage: `name` , `accepted_types` , `compute_schema` , `supports_cpp_node` e `_build_single` . Na prática é necessário definir mais um método que executará o trabalho real do estágio, por convenção se chama `on_data` .

obs.: é necessário colocar a anotação @property antes de inserir esses métodos

=== "name"
    Utilizada para retornar um nome mais amigável para o estágio, usada apenas para fins de depuração.
    ```py
    def name(self) -> str:
        return "pass-thru"
    ```
=== "accepted_types"
    Retorna uma tupla de classes de mensagem que este estágio é capaz de aceitar como entrada. (Permite que o Morpheus valide se a saída do pai desse estágio é o mesmo da entrada).
    ```py
    def accepted_types(self) -> tuple:
        return (typing.Any,)
    ```
=== "compute_schema"
    Retorna o tipo de saída do estágio (Como vamos herdar da classe ``PassThruTypeMixin``, não precisamos utiliza-la):
    ```py
    def compute_schema(self, schema: StageSchema):
        schema.output_schema.set_type(schema.input_type)
    ```
=== "supports_cpp_node"
    Retorna se estamos utilizando arquivos CPP nesse estágio.
    ```py
    def supports_cpp_node(self) -> bool:
        return False
    ```
=== "_build_single"
    Método que será usado no momento de construção do estágio, para construir o nó e conecta-lo ao pipeline, ele recebe uma instância de um construtor do mrc(Morpheus Runtime Core) do tipo ``mrc.Builder``, junto com um nó de entrada do tipo ``SegmentObject`` e retorna um nó recém construído.
    ```py
    def _build_single(self, builder: mrc.Builder, input_node: mrc.SegmentObject) -> mrc.SegmentObject:
        node = builder.make_node(self.unique_name, ops.map(self.on_data))
        builder.make_edge(input_node, node)

        return node
    ```

    Linha a linha:

    `node = builder.make_node(self.unique_name, ops.map(self.on_data))`
    
    Cria um nó com o nome que demos para a classe junto a um ID único para que não seja possível existir estágios com mesmo nome;

    `builder.make_edge(input_node, node)`

    Define uma aresta conectando o novo nó ao nó pai.


=== "on_data"
    Aceita uma mensagem recebida e retorna uma mensagem (lógica do estágio)
    ```py
    def on_data(self, message: typing.Any):
        # Retorna mensagem para o proximo estágio
        return message
    ```

---

A Classe completa ficaria assim:

```py
import typing

import mrc
from mrc.core import operators as ops

from morpheus.cli.register_stage import register_stage
from morpheus.pipeline.pass_thru_type_mixin import PassThruTypeMixin
from morpheus.pipeline.single_port_stage import SinglePortStage


@register_stage("pass-thru")
class PassThruStage(PassThruTypeMixin, SinglePortStage):

    @property
    def name(self) -> str:
        return "pass-thru"

    def accepted_types(self) -> tuple:
        return (typing.Any, )

    def supports_cpp_node(self) -> bool:
        return False

    def on_data(self, message: typing.Any):
        # Return the message for the next stage
        return message

    def _build_single(self, builder: mrc.Builder, input_node: mrc.SegmentObject) -> mrc.SegmentObject:
        node = builder.make_node(self.unique_name, ops.map(self.on_data))
        builder.make_edge(input_node, node)

        return node
```

---

## Criação de um Pipeline simples

Antes de construir o pipeline é preciso configurar o ambiente, começando com o log:

```py
configure_logging(log_level=logging.DEBUG)
```

Os manipuladores de registro não são bloqueantes, pois utilizam uma fila para enviar as mensagens de registro em um thread separado.

!!! note
    Podemos utilizar a função `configure_logging` para adicionar um ou mais manipuladores de registro à configuração padrão, um exemplo disso seria:

    ```python
    loki_handler = logging_loki.LokiHandler(
        url=f"{loki_url}/loki/api/v1/push",
        tags={"app": "morpheus"},
        version="1",
    )

    configure_logging(loki_handler, log_level=log_level)
    ```

    Onde é adicionado um manipulador de log chamado Grafana Loki que cria logs únicos para depuração posteriormente.

Após configurarmos o Log, precisamos criar uma instância de configuração (sempre será necessário):

```py
config = Config()
```

Para o exemplo será utilizado a classe `FileSourceStage` , para ler um arquivo grande no qual cada linha é um objeto JSON. O estágio pegará essas linhas e as empacotará como objetos de mensagem Morpheus para nosso estágio de passagem consumir.

```py
pipeline.set_source(FileSourceStage(config, filename=input_file, iterative=False))
```

Em Seguida é adicionado os estágios ao pipeline, junto com a instância ``MonitorStage`` , para monitorar a taxa de transferência dos estágios:

```py
# Adiciona a função de estágio com o decorador @stage
pipeline.add_stage(pass_thru_stage(config))

# Adiciona o monitoramento de performance do estágio anterior
pipeline.add_stage(MonitorStage(config))

# Adiciona a classe criada como estágio
pipeline.add_stage(PassThruStage(config))

# Adiciona o monitoramento de performance do estágio anterior
pipeline.add_stage(MonitorStage(config))
```

### Pipeline Completo

```py
import logging
import os

from pass_thru import PassThruStage
from pass_thru_deco import pass_thru_stage

from morpheus.config import Config
from morpheus.pipeline import LinearPipeline
from morpheus.stages.general.monitor_stage import MonitorStage
from morpheus.stages.input.file_source_stage import FileSourceStage
from morpheus.utils.logger import configure_logging

def run_pipeline():
    # Configura o Log do Morpheus
    configure_logging(log_level=logging.DEBUG)

    root_dir = os.environ['MORPHEUS_ROOT']
    input_file = os.path.join(root_dir, 'examples/data/email_with_addresses.jsonlines')

    config = Config()

    # Cria um Objeto para pipeline linear (a saida de um estágio é a entrada da outra na ordem que foi criada)
    pipeline = LinearPipeline(config)

    # Faz a leitura do arquivo JSON
    pipeline.set_source(FileSourceStage(config, filename=input_file, iterative=False))

    # Adiciona a função de estágio com o decorador @stage
    pipeline.add_stage(pass_thru_stage(config))

    # Adiciona o monitoramento de performance do estágio anterior
    pipeline.add_stage(MonitorStage(config))

    # Adiciona a classe criada como estágio
    pipeline.add_stage(PassThruStage(config))

    # Adiciona o monitoramento de performance do estágio anterior
    pipeline.add_stage(MonitorStage(config))

    pipeline.run()

if __name__ == "__main__":
    run_pipeline()
```

