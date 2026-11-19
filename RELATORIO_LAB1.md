# Relatório Técnico — Lab I: Worm na Rede

## 1. Objetivo

Analisar uma captura de tráfego de rede (PCAPNG) para identificar comportamentos anômalos, investigar possíveis tentativas de propagação via SMB (TCP/445) e priorizar o principal host para investigação.

A análise busca caracterizar o tráfego observado. Ela não presume que um host esteja infectado apenas por apresentar comportamento anômalo.

## 2. Ambiente e metodologia

- **Sistema operacional:** Windows
- **Ambiente de análise:** VS Code e Jupyter Notebook
- **Bibliotecas:** Scapy, Pandas, NumPy, Matplotlib e Scikit-learn
- **Arquivo analisado:** `pacotes_worm.pcapng`
- **Técnicas:** inspeção de pacotes, agregação de características por IP de origem, contagem de SYNs, análise temporal, z-scores e Isolation Forest.

A captura foi analisada sem executar malware.

### Características usadas no Isolation Forest

O modelo foi treinado com 12 características numéricas agregadas por IP:

| Característica | Significado |
|---|---|
| `pkt_count` | Quantidade total de pacotes observados com aquele IP como origem |
| `byte_count` | Soma dos tamanhos dos pacotes originados pelo IP |
| `unique_dst` | Quantidade de endereços IP de destino distintos |
| `pkt_count_on_target` | Pacotes originados pelo IP com destino à porta TCP/445 |
| `byte_count_on_target` | Soma dos tamanhos desses pacotes destinados à porta TCP/445 |
| `unique_dst_on_target` | Destinos IP distintos nos pacotes destinados à porta TCP/445 |
| `syn_count` | Pacotes TCP SYN sem ACK originados pelo IP, para quaisquer portas de destino |
| `rst_count` | Pacotes TCP com a flag RST originados pelo IP |
| `duration_s` | Intervalo entre o primeiro e o último pacote observado para o IP |
| `pps` | Média de pacotes por segundo no intervalo observado |
| `pps_target` | Média de pacotes destinados à porta TCP/445 por segundo no intervalo observado |
| `fraction_to_target` | Proporção de pacotes originados pelo IP cujo destino é a porta TCP/445 |

As taxas por segundo são médias calculadas sobre o intervalo entre o primeiro e o último pacote observado para cada origem; não representam necessariamente a taxa instantânea em cada segundo.

### Configuração do modelo

Foi usado `StandardScaler` antes do `IsolationForest`, configurado com `n_estimators=200`, `contamination="auto"` e `random_state=42`. O modelo recebeu as 12 características listadas acima.

O campo `anomaly_score` do notebook vem de `modelo.score_samples(X_scaled)`: quanto menor o valor, mais anômalo é o registro segundo a função de pontuação do modelo. Já `anomaly_label` vem de `modelo.predict()`: `-1` significa classificado como anomalia e `1`, não classificado como anomalia. Essas classificações não equivalem a rótulos de malware.

## 3. Visão geral da captura

| Indicador | Resultado |
|---|---:|
| Total de pacotes | 206.536 |
| Pacotes TCP | 182.823 |
| Pacotes envolvendo a porta 443 | 19.456 |
| Pacotes envolvendo a porta 445 | 2.014 |
| IPs de origem analisados | 79 |

As contagens de pacotes envolvendo as portas 443 e 445 incluem tráfego com a porta correspondente na origem ou no destino. Por isso, não devem ser comparadas diretamente com contagens restritas a pacotes cujo destino é a porta 445.

## 4. Detecção de anomalias

O Isolation Forest analisou 79 IPs e classificou 8 como anomalias.

- **Principal IP para investigação:** `172.19.15.200`
- **`anomaly_score` (`score_samples`):** `-0,868919`
- **Rótulo do modelo:** `-1` (anomalia)

Esse foi o menor score entre os IPs analisados, tornando `172.19.15.200` o registro mais anômalo segundo o modelo utilizado. Isso não comprova, por si só, atividade maliciosa: serviços legítimos, gateways e outros equipamentos também podem se comportar de forma estatisticamente atípica.

## 5. Evidências relacionadas ao IP suspeito

O endereço `172.19.15.200` apresentou os seguintes indicadores:

| Indicador | Resultado |
|---|---:|
| Pacotes originados pelo IP | 30.219 |
| Destinos únicos em geral | 513 |
| Pacotes originados pelo IP com destino à porta TCP/445 | 1.920 |
| Destinos únicos nesses pacotes para TCP/445 | 269 |
| SYNs sem ACK destinados à porta TCP/445 | 1.912 |
| SYNs sem ACK para quaisquer portas TCP de destino | 2.396 |
| Primeiro SYN observado para a porta 445 | 2025-11-02 19:33:31.212 UTC |

A distinção entre os contadores é importante: os 2.396 SYNs são a contagem geral de SYNs originados pelo IP para quaisquer portas de destino; os 1.912 SYNs são especificamente os pacotes direcionados à porta TCP/445. Da mesma forma, os 1.920 pacotes para TCP/445 incluem outros tipos de pacotes além dos SYNs.

A atividade de SYNs destinados à porta 445 foi observada em 49 minutos distintos. O pico registrado foi de 110 SYNs em um minuto ativo, com média de aproximadamente 39,0 SYNs por minuto ativo (1.912 divididos pelos 49 minutos com atividade).

O notebook identificou 289 tuplas distintas de SYN e 269 destinos IP distintos. A diferença indica que houve mais de uma tupla associada a alguns destinos. A proporção de pacotes que excede a primeira ocorrência de cada tupla foi calculada como `(1.912 - 289) / 1.912`, aproximadamente 84,9%. Esse valor **não** prova que todos esses pacotes sejam retransmissões: pode incluir tentativas repetidas, retransmissões ou efeitos da coleta/captura.

A análise dos SYNs destinados à porta TCP/445 identificou `172.19.15.200` como a única origem iniciadora desse tipo de tráfego na captura examinada. Foram observadas respostas SYN-ACK de alguns destinos, o que demonstra resposta ao início de conexão, mas não comprova exploração SMB bem-sucedida.

## 6. Análise estatística

Os z-scores calculados para `172.19.15.200` mostram desvios expressivos em relação aos demais IPs:

| Característica | Z-score |
|---|---:|
| `byte_count_on_target` | +8,832 |
| `pps_target` | +8,832 |
| `pkt_count_on_target` | +8,832 |
| `fraction_to_target` | +8,832 |
| `unique_dst_on_target` | +8,832 |
| `syn_count` | +8,819 |
| `unique_dst` | +8,799 |
| `pkt_count` | +2,738 |
| `duration_s` | +1,689 |
| `rst_count` | +0,796 |
| `pps` | −0,192 |
| `byte_count` | +0,132 |

Os cinco z-scores iguais a `+8,832` resultam da distribuição observada dessas características, em que os demais IPs têm valores zero ou praticamente todos zero. Com uma única observação não nula e as demais iguais a zero, diferentes características podem produzir o mesmo z-score. Portanto, esses cinco valores não devem ser tratados como cinco evidências independentes.

A taxa global média de pacotes por segundo (`pps`) ficou próxima da média dos IPs analisados. Isso indica que, neste conjunto de características, a diversidade de destinos e a concentração do tráfego na porta 445 distinguem o host mais claramente do que sua taxa global média de pacotes.

## 7. Cronologia e hipótese de propagação

O primeiro pacote de qualquer tipo atribuído a `172.19.15.200` foi observado às 19:11:37.607 UTC. O primeiro SYN destinado à porta TCP/445 foi observado às 19:33:31.212 UTC.

Outros IPs classificados como anomalias já apresentavam tráfego geral antes do primeiro pacote observado de `172.19.15.200`. Isso não demonstra que esses outros hosts estivessem infectados: o primeiro pacote de uma origem na captura não é necessariamente o início de uma atividade maliciosa.

O padrão de SYNs para muitos destinos é compatível com varredura SMB automatizada. A hipótese de tentativa de propagação de worm é plausível, mas a captura não permite determinar, por si só, se houve exploração bem-sucedida ou qual host foi infectado primeiro.

O notebook também apresenta uma distribuição por sub-rede dos destinos (`172.15.19.0/24` e `172.19.15.0/24`). Essa distribuição deve ser validada contra os pacotes e o contexto de endereçamento antes de ser usada para conclusões sobre os alvos; endereços dessas duas redes não são equivalentes.

## 8. Conclusão

O endereço `172.19.15.200` foi identificado como o principal host suspeito para investigação na captura analisada. Essa priorização é sustentada pela combinação de:

- menor `score_samples` entre os IPs analisados e classificação como anomalia pelo Isolation Forest;
- 1.912 pacotes SYN sem ACK destinados à porta TCP/445;
- 269 destinos IP distintos associados aos pacotes destinados à porta TCP/445;
- desvios estatísticos elevados nas características relacionadas à porta 445 e à diversidade de destinos;
- padrão de tentativas observado ao longo de 49 minutos ativos.

O conjunto de evidências é compatível com varredura SMB automatizada e justifica investigação adicional sobre uma possível tentativa de propagação de worm. Entretanto, a captura não comprova execução de malware, exploração bem-sucedida ou comprometimento dos destinos. A identificação do paciente zero permanece inconclusiva.

## 9. Recomendações

Em um ambiente real, recomenda-se:

1. Preservar a captura original e os registros relevantes, mantendo sua integridade.
2. Identificar o equipamento correspondente ao IP `172.19.15.200` no horário da captura.
3. Correlacionar os horários com logs de firewall, endpoints, autenticação e serviços SMB.
4. Investigar respostas SYN-ACK, estabelecimento de conexões, tráfego SMB subsequente e eventuais indícios de exploração.
5. Validar a distribuição dos destinos por sub-rede diretamente na captura.
6. Se a atividade estiver ocorrendo em produção, avaliar medidas de contenção conforme o procedimento de resposta a incidentes da organização.

## 10. Limitações

- A análise depende do conteúdo e da visibilidade da captura; tráfego ausente ou perdido pode alterar as conclusões.
- Contagens de pacotes não equivalem necessariamente a conexões independentes. Pacotes repetidos podem representar retransmissões, tentativas repetidas ou efeitos da coleta.
- O Isolation Forest identifica desvios estatísticos, não intenção maliciosa. Não foi realizada validação supervisionada contra rótulos conhecidos.
- Não foram apresentados registros de endpoint que comprovem execução de malware nem evidências conclusivas de exploração SMB ou comprometimento dos destinos.
- A distribuição por sub-rede precisa ser validada antes de sustentar conclusões específicas sobre os alvos.
- A identificação do paciente zero permanece inconclusiva.

---

**Status do laboratório:** análise estatística e investigação inicial concluídas. Principal host suspeito priorizado para investigação; confirmação de infecção, exploração e paciente zero depende de evidências adicionais.

---

## 11. Créditos e referências

### Laboratório original

Laboratório e arquivo de captura utilizados como base para este estudo:

- **Repositório:** [hackone_pos — Carlos Silva](https://github.com/carlossilva-cybersec/hackone_pos)
- **Diretório do laboratório:** `network-anomaly-detection-lab/`
- **Material utilizado:** enunciado do Lab I e captura `pacotes_worm.pcapng`.

O repositório original é a fonte do material de referência e da captura utilizada. Este relatório documenta a análise desenvolvida durante a execução do laboratório; não é uma publicação oficial do autor do repositório. Verifique a licença e as permissões aplicáveis antes de redistribuir materiais ou capturas.

### Ferramentas e documentação técnica

- [Scapy](https://scapy.readthedocs.io/) — leitura e análise de pacotes.
- [Pandas](https://pandas.pydata.org/docs/) — tratamento e organização dos dados.
- [NumPy](https://numpy.org/doc/) — operações numéricas.
- [Matplotlib](https://matplotlib.org/stable/) — geração de visualizações.
- [Scikit-learn — Isolation Forest](https://scikit-learn.org/stable/modules/generated/sklearn.ensemble.IsolationForest.html) — detecção não supervisionada de anomalias.

**Nota:** este projeto tem finalidade educacional e defensiva. As conclusões devem ser interpretadas dentro dos limites da captura e da metodologia descrita.
