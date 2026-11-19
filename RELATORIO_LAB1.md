# Relatório Técnico — Lab I: Worm na Rede

## 1. Objetivo

Analisar uma captura de tráfego de rede (PCAPNG) para identificar comportamentos anômalos, investigar possíveis tentativas de propagação via SMB (TCP/445) e determinar o principal host suspeito.

## 2. Ambiente e metodologia

- **Sistema operacional:** Windows
- **Ambiente de análise:** VS Code e Jupyter Notebook
- **Bibliotecas:** Scapy, Pandas, NumPy e Scikit-learn
- **Arquivo analisado:** `pacotes_worm.pcapng`
- **Técnicas:** análise de pacotes, extração de características por IP, Isolation Forest e cálculo de z-scores.

A análise foi realizada sobre a captura de rede, sem executar malware.

## 3. Visão geral da captura

| Indicador | Resultado |
|---|---:|
| Total de pacotes | 206.536 |
| Pacotes TCP | 182.823 |
| Pacotes envolvendo a porta 443 | 19.456 |
| Pacotes envolvendo a porta 445 | 2.014 |
| IPs de origem analisados | 79 |

Os números de portas incluem pacotes que envolvem a porta correspondente na origem ou no destino, conforme a contagem realizada.

## 4. Detecção de anomalias

Foi aplicado o algoritmo Isolation Forest sobre 12 características numéricas de tráfego por IP.

- IPs analisados: 79
- IPs classificados como anomalias: 8
- Principal anomalia: `172.19.15.200`
- Score do modelo: `-0,868919`

O score mais baixo entre os IPs analisados indica que esse endereço apresenta o comportamento mais anômalo segundo o modelo utilizado. A classificação estatística, isoladamente, não comprova atividade maliciosa.

## 5. Evidências relacionadas ao IP suspeito

O endereço `172.19.15.200` apresentou:

| Indicador | Resultado |
|---|---:|
| Pacotes enviados pela origem | 30.219 |
| Destinos únicos em geral | 513 |
| Pacotes com destino à porta TCP/445 | 1.920 |
| Destinos únicos na porta TCP/445 | 269 |
| SYNs destinados à porta TCP/445 | 1.912 |
| SYNs para diferentes portas TCP | 2.396 |
| Primeiro SYN para a porta 445 | 2025-11-02 19:33:31 UTC |

A atividade relacionada à porta 445 foi observada em 49 minutos distintos. Em vários minutos, foram registrados 42 SYNs para 7 destinos diferentes. Também foram observadas combinações TCP repetidas, que podem representar retransmissões, novas tentativas ou duplicações na captura.

A análise específica dos SYNs para TCP/445 identificou apenas `172.19.15.200` como origem iniciadora dessas conexões na captura examinada.

## 6. Análise estatística

Os z-scores do host suspeito indicaram desvios expressivos em relação aos demais IPs:

| Característica | Z-score |
|---|---:|
| Pacotes destinados à porta 445 | +8,832 |
| Destinos únicos na porta 445 | +8,832 |
| SYNs TCP | +8,819 |
| Destinos únicos em geral | +8,799 |
| Total de pacotes | +2,738 |

Os valores iguais de z-score para algumas características relacionadas à porta 445 decorrem da distribuição dos dados, na qual os demais IPs apresentaram valores zero nessas métricas. Portanto, esses resultados não devem ser considerados evidências independentes.

A taxa global de pacotes por segundo do host não se destacou significativamente. Isso sugere que a diversidade de destinos e o padrão de conexões são mais relevantes para caracterizar a anomalia do que o volume global isolado.

## 7. Cronologia e hipótese de propagação

O primeiro pacote geral de `172.19.15.200` foi observado às 19:11:37 UTC. Entretanto, o primeiro SYN para TCP/445 ocorreu às 19:33:31 UTC.

Outros IPs anômalos já apresentavam tráfego geral antes desse horário. Isso não comprova que algum deles estivesse infectado, pois o primeiro pacote observado não equivale necessariamente ao início de uma atividade maliciosa.

A captura sustenta a hipótese de que `172.19.15.200` realizou varredura SMB ou tentativas automatizadas de conexão. Não permite determinar, por si só, se houve exploração bem-sucedida ou qual host foi infectado primeiro.

## 8. Conclusão

O endereço `172.19.15.200` foi identificado como o principal host suspeito de atividade ofensiva na captura analisada.

A conclusão é sustentada pela combinação de:

- Classificação como a anomalia mais significativa pelo Isolation Forest.
- 1.912 SYNs destinados à porta TCP/445.
- 269 destinos distintos associados a esse tráfego.
- Desvios estatísticos expressivos nas características de SMB e diversidade de destinos.
- Padrão repetitivo de conexões observado ao longo da captura.

O comportamento é compatível com varredura SMB automatizada e possível tentativa de propagação de worm. Contudo, não foi comprovada a infecção do host, a exploração de vulnerabilidades nem a identidade do paciente zero.

## 9. Recomendações

Em um ambiente real, recomenda-se:

1. Preservar a captura e os registros relevantes.
2. Identificar o equipamento correspondente ao IP `172.19.15.200`.
3. Correlacionar os horários com logs de firewall, endpoints e serviços SMB.
4. Investigar as conexões, respostas e eventuais indícios de exploração.
5. Se a atividade estiver ocorrendo em produção, avaliar contenção conforme o procedimento de resposta a incidentes.

## 10. Limitações

A análise depende do conteúdo e da visibilidade da captura. Não foram apresentados, nesta etapa, registros de endpoint que comprovem execução de malware, evidências conclusivas de exploração SMB ou confirmação de comprometimento dos destinos.

A identificação de paciente zero permanece inconclusiva.

---

**Status do laboratório:** análise estatística e investigação inicial concluídas. Principal host suspeito identificado; confirmação de infecção e paciente zero pendentes de evidências adicionais.

---

## 11. Créditos e referências

### Laboratório original

Laboratório e arquivos de captura utilizados como base para este estudo:

- **Repositório:** [hackone_pos — Carlos Silva](https://github.com/carlossilva-cybersec/hackone_pos)
- **Diretório do laboratório:** `network-anomaly-detection-lab/`
- **Material utilizado:** enunciado do Lab I e captura de tráfego `pacotes_worm.pcapng`.

O repositório original é a fonte do material de referência e da captura utilizada. A análise, a implementação das etapas de processamento, a aplicação do modelo de detecção de anomalias e as conclusões deste relatório foram desenvolvidas durante a execução deste laboratório.

### Ferramentas e documentação técnica

- [Scapy](https://scapy.readthedocs.io/) — leitura e análise de pacotes.
- [Pandas](https://pandas.pydata.org/docs/) — tratamento e organização dos dados.
- [Scikit-learn — Isolation Forest](https://scikit-learn.org/stable/modules/generated/sklearn.ensemble.IsolationForest.html) — detecção não supervisionada de anomalias.

**Nota:** este projeto é um estudo educacional independente, baseado no material referenciado acima, e não deve ser interpretado como uma publicação oficial do autor do repositório.
