# Inatel IC - IA e Cybersecurity

Repositório voltado para documentação dos estudos da Iniciação Científica com foco em Inteligência Artificial aplicada à segurança cibernética,
utilizando a plataforma NVIDIA Morpheus para detecção e resposta a ameaças em tempo real.

Documentação: [fiddelis.me/inatel-ic-morpheus](http://fiddelis.me/inatel-ic-morpheus/)

## Assuntos Estudados

### Morpheus

O [Morpheus](https://developer.nvidia.com/morpheus-cybersecurity) é uma Framework de segurança cibernética da NVIDIA de código aberto, **acelerada por GPU/DPU**, que permite a criação de pipelines otimizados para filtrar, processar e classificar um grande volume de dados na rede em **tempo real**.

<p align="center">
  <img width=600 src=https://external-content.duckduckgo.com/iu/?u=https%3A%2F%2Ftse1.mm.bing.net%2Fth%3Fid%3DOIP.794Bd8ghraH46XnFi1kiJwHaD4%26pid%3DApi&f=1&ipt=76d613a56414ddebc7521a01d79f9886d32723594728d51a37acf918e6a26748&ipo=images>
</p>

Utiliza **Machine Learning** para identificar, capturar e agir sobre as ameaças.

#### Benefícios

- Permite que desenvolvedores **criem suas próprias soluções** de segurança cibernética.
- Permite a **adaptação dinâmica** a feedback humano e eventos externos, melhorando a precisão e a capacidade de resposta dos modelos de IA.
- Possível Implementar o próprio modelo ou alguns dos modelos pré-treinados e testados pela própria NVIDIA.

### Triton Inferênce Server

O [Triton Inference Server](https://developer.nvidia.com/triton-inference-server) fornece uma solução de inferência de nuvem e borda otimizada por CPUs e GPUs. O Triton suporta um protocolo HTTP/REST e GRPC que permite que clientes remotos solicitem inferência para qualquer modelo gerenciado pelo servidor.

<p align="center">
  <img width=600 src=https://external-content.duckduckgo.com/iu/?u=https%3A%2F%2Fi.ytimg.com%2Fvi%2FNQDtfSi5QF4%2Fmaxresdefault.jpg%3Fsqp%3D-oaymwEmCIAKENAF8quKqQMa8AEB-AH-CYAC0AWKAgwIABABGDIgQCh_MA8%3D%26rs%3DAOn4CLCStCtyFbCy6h_cLxxfd2c6UPdijg&f=1&nofb=1&ipt=7f96467636618dda2de08bba86adae22c2f7fd91c7f420a61d1e5fbb6faf452b&ipo=images>
</p>
