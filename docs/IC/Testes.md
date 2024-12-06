# Arquitetura para teste

Transformando Wazuh de um SIEM/IDR para um XDR com Morpheus

## Ferramentas

- Pfsense (Firewall)
- Snort (IDS/IPS)
- Caldera (Simulação de ataques)
  - Se baseia no Mitre Att&ck que é um banco de dados que é atualizado constantemente que contem TTP's de inumeros invasores.
- Wazuh (SIEM/IDR)
- Morpheus (Inferencias)

## Métricas para avaliação

1. Taxa de detecção
2. Taxa de falso positivo
3. Tempo de resposta
4. Uso de recursos