# Triton Server atráves do Docker

Muitos casos de testes **onde é necessário a analise dos dados a partir de uma IA** é preciso que tenha em execução o Triton Inference Server, ==é ele quem cuida da classificação dos dados== a partir do seu modelo pré-treinado.

No momento que estou escrevendo esse documento, o Triton Inference Server se encontra na versão ^^24.09^^.

---

## Pré-requisitos

- [Docker](https://docs.docker.com/get-docker/)
- [The NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html#docker)
---

## Download da Imagem
Com o Docker ja instalado na sua maquina, precisamos baixar a imagem na nuvem da NVIDIA ([NGC](https://catalog.ngc.nvidia.com/orgs/nvidia/containers/tritonserver)).

```sh
docker pull nvcr.io/nvidia/tritonserver:24.09-trtllm-python-py3
```

Para verificar se o comando funcionou corretamente, execute o comando abaixo para listar todas as imagens instaladas no Docker:

```sh
docker ps
```

??? failure
    Caso a imagem não se encontre na lista, verifique se inseriu a tag corretamente e tente novamente.

---

## Executando o Triton Server

A execução do Triton Server depende do tipo do modelo que será executado, oque varia de projeto para projeto.

^^Exemplo de execução:^^

```sh
docker run --rm -ti --gpus=all -p8000:8000 -p8001:8001 -p8002:8002\
    -v $PWD/models:/models\
    nvcr.io/nvidia/tritonserver:23.06-py3 tritonserver\
    --model-repository=/models/triton-model-repo\
    --exit-on-error=false\
    --log-info=true\
    --strict-readiness=false\
    --disable-auto-complete-config
```

Esse comando faz com que seja executado um cantainer docker do Triton que execute todos os modelos na pasta de repositório especificada.

!!! note
    Como não foi especificado nenhum modelo, o Triton Server irá **executar todos os modelos** que encontrar na pasta de repositório.

    Para especificar o modelo utilize o comando:
    ```sh
    --load-model {{NOME-DO-MODELO}}
    ```

Depois que o Triton carregar o modelo, será exibido algo parecido com:

```sh

+-------------------+---------+--------+
| Model             | Version | Status |
+-------------------+---------+--------+
| {NOME-DO-MODELO}  | 1       | READY  |
+-------------------+---------+--------+
```
??? failure
    Se isso não estiver presente na saída, verifique o log do Triton para quaisquer mensagens de erro relacionadas ao carregamento do modelo.