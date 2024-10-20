# Pipelines com Python

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

