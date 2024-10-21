# Pipelines com CLI

As criações de pipelines do Morpheus podem ser feitas atráves do CLI, onde os estágios são ==executados de forma linear==, significando que a saída de um estágio é a entrada do próximo.

!!! note
    Existem diferentes tipos de comandos possiveis no Morpheus, para saber mais sobre cada uma delas, verifique utilizando:

    ```sh
    morpheus run --help
    ```

    Ou na própria [documentação do Morpheus](https://docs.nvidia.com/morpheus/basics/overview.html).

---

## Exemplo

Para explicar o funcionamento a partir do CLI, será utilizado um exemplo de execução de uma Pipeline para ^^detecção de mineração de criptomoedas na GPU^^.

!!! warning
    Para a execução desse exemplo, é necessário que você tenha clonado o repositório do Morpheus;

    ```sh
    MORPHEUS_ROOT=$(pwd)/morpheus
    git clone https://github.com/nv-morpheus/Morpheus.git $MORPHEUS_ROOT
    cd $MORPHEUS_ROOT
    ```

    e que tenha executado o script para o download dos arquivos maiores.

    ```sh
    scripts/fetch_data.py fetch examples models
    ```

    ou se preferir, verifique a documentação do Morpheus na sessão [Git LFS](https://docs.nvidia.com/morpheus/getting_started.html#git-lfs).

---

## Executando Triton

Antes que você execute o Pipeline no Morpheus, é necessário que você inicialize o modelo com o Triton Server.

```sh
docker run --rm -ti --gpus=all -p8000:8000 -p8001:8001 -p8002:8002\
    -v $PWD/models:/models nvcr.io/nvidia/tritonserver:23.06-py3 tritonserver\
        --model-repository=/models/triton-model-repo\
        --exit-on-error=false\
        --model-control-mode=explicit\
        --load-model abp-nvsmi-xgb
```

obs.: ^^Esse comando considera que você esteja na pasta root do Morpheus.^^

Ao subir o modelo, é preciso que tenha exibido o seguinte resultado:

```sh
+-------------------+---------+--------+
| Model             | Version | Status |
+-------------------+---------+--------+
| abp-nvsmi-xgb     | 1       | READY  |
+-------------------+---------+--------+
```

---

## Executando o Morpheus

O código a ser executado nesse exemplo será:

```sh
morpheus --log_level=DEBUG \
    run --num_threads=8 --pipeline_batch_size=1024 --model_max_batch_size=1024 \
    pipeline-fil --columns_file=data/columns_fil.txt \
    from-file --filename=examples/data/nvsmi.jsonlines \
    deserialize \
    preprocess \
    inf-triton --model_name=abp-nvsmi-xgb --server_url=localhost:8000 \
    monitor --description "Inference Rate" --smoothing=0.001 --unit inf \
    add-class \
    serialize --include 'mining' \
    to-file --filename=detections.jsonlines --overwrite
```

Para o entendimento, iremos analisar linha a linha do comando.

```sh
# Define o nivel do log para o modo de Debug

morpheus --log_level=DEBUG \
```

```sh
# Executa um pipeline com 8 threads e um tamanho de lote de modelo de 1024
# (deve ser igual ou menor que a configuração do Triton)

run --num_threads=8 --pipeline_batch_size=1024 --model_max_batch_size=1024 \
```

```sh
# Especifica um pipeline FIL com 256 de comprimento de sequência
# (deve corresponder à configuração do Triton)

pipeline-fil --columns_file=data/columns_fil.txt \
```

```sh
# Faz a leitura de um arquivo na pasta especificada

from-file --filename=examples/data/nvsmi.jsonlines \
```

```sh
# Disserializa o arquivo para poder ser processado

deserialize \
```

```sh
# Pré-processamento que converte as entradas para um token BERT

preprocess \
```

```sh
# Envia mensagem para o Triton fazer a inferencia
# (é necessário especificar o modelo)

inf-triton --model_name=abp-nvsmi-xgb --server_url=localhost:8000 \
```

```sh
# Monitora o estágio anterior e exibe no console as informações de performance

monitor --description "Inference Rate" --smoothing=0.001 --unit inf \
```

```sh
# Adiciona o resultado de inferência na mensagem

add-class \
```

```sh
# Converte a mensagem de Objeto para String novamente,
# contendo somente as informações resultantes da inferência

serialize --include 'mining' \
```

```sh
# Faz a escrita do resultado em um arquivo .jsonline

to-file --filename=detections.jsonlines --overwrite
```

A ordem de execução do Pipeline foi linear, onde a saída do nó anterior vai para a entrada do próximo nó.

---

## Resultado

Ao concluir a execução do Pipeline, teremos os seguintes resultados.

### Console

```sh
...
====Registering Pipeline====
====Registering Pipeline Complete!====
====Starting Pipeline====
====Pipeline Started====
====Building Pipeline====
Added source: <from-file-0; FileSourceStage(filename=examples/data/nvsmi.jsonlines, iterative=False, file_type=FileTypes.Auto, repeat=1, filter_null=True)>
  └─> morpheus.MessageMeta
Added stage: <deserialize-1; DeserializeStage()>
  └─ morpheus.MessageMeta -> morpheus.MultiMessage
Added stage: <preprocess-fil-2; PreprocessFILStage()>
  └─ morpheus.MultiMessage -> morpheus.MultiInferenceFILMessage
Added stage: <inference-3; TritonInferenceStage(model_name=abp-nvsmi-xgb, server_url=localhost:8000, force_convert_inputs=False, use_shared_memory=False)>
  └─ morpheus.MultiInferenceFILMessage -> morpheus.MultiResponseMessage
Added stage: <monitor-4; MonitorStage(description=Inference Rate, smoothing=0.001, unit=inf, delayed_start=False, determine_count_fn=None)>
  └─ morpheus.MultiResponseMessage -> morpheus.MultiResponseMessage
Added stage: <add-class-5; AddClassificationsStage(threshold=0.5, labels=[], prefix=)>
  └─ morpheus.MultiResponseMessage -> morpheus.MultiResponseMessage
Added stage: <serialize-6; SerializeStage(include=['mining'], exclude=['^ID$', '^_ts_'], fixed_columns=True)>
  └─ morpheus.MultiResponseMessage -> morpheus.MessageMeta
Added stage: <to-file-7; WriteToFileStage(filename=detections.jsonlines, overwrite=True, file_type=FileTypes.Auto)>
  └─ morpheus.MessageMeta -> morpheus.MessageMeta
====Building Pipeline Complete!====
Starting! Time: 1656353254.9919598
Inference Rate[Complete]: 1242inf [00:00, 1863.04inf/s]
====Pipeline Complete====
```

!!! note
    Como utilizamos a configuração de Debug no Log, tivemos informações de tudo que estáva acontecendo na criação dos estágios.

### Arquivo

```json
...
{"mining": 0}
{"mining": 0}
{"mining": 0}
{"mining": 0}
{"mining": 1}
{"mining": 1}
{"mining": 1}
{"mining": 1}
{"mining": 1}
{"mining": 1}
{"mining": 1}
{"mining": 1}
...
```

!!! note
    Foi configurado no estágio para ser gerado apenas o resultado da classificação da inferência.

---

## Executando Estágios em Python com CLI

Como alternativa ao Pipeline com Python, poderíamos testar o estágio baseado em classe em um pipeline construído usando a ferramenta de linha de comando Morpheus. Precisaremos passar o caminho para nosso estágio por meio do argumento `--plugin` para que ele fique visível para a ferramenta de linha de comando.

!!! warning
    Até o momento só é possível registar o estágio feito em classes.

```sh
morpheus --log_level=debug --plugin examples/developer_guide/1_simple_python_stage/pass_thru.py \
  run pipeline-other \
  from-file --filename=examples/data/email_with_addresses.jsonlines \
  pass-thru \
  monitor
```