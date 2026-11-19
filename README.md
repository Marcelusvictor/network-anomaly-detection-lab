# Network Anomaly Detection Lab — Worm na Rede

## Overview

Educational cybersecurity lab focused on network traffic analysis and
anomaly detection using Python.

The project investigates suspicious TCP traffic targeting port 445
(SMB), identifies anomalous source IP addresses, and analyzes SYN
connection attempts to detect behavior consistent with automated scanning.

## Objectives

- Analyze TCP traffic and packets involving ports 443 and 445.
- Extract traffic features by source IP address.
- Apply Isolation Forest to identify anomalous hosts.
- Investigate SYN attempts targeting TCP port 445.
- Evaluate statistical deviations using Z-scores.
- Document findings, limitations, and defensive recommendations.

## Tools

- Python
- Scapy
- Pandas
- Scikit-learn
- Jupyter Notebook

## Project files

- `Worm_445_Analise.ipynb` — analysis notebook, calculations, and results.
- `RELATORIO_LAB1.md` — technical report and conclusions.
- `requirements.txt` — Python dependencies.

## Main finding

The host `172.19.15.200` generated 1,912 observed SYN packets
destined for TCP port 445, targeting 269 distinct destination IPs.
Its traffic pattern was consistent with automated SMB scanning.

The findings indicate suspicious scanning behavior in the analyzed
capture. They do not, by themselves, prove successful exploitation,
malware infection, or the identity of the initial compromised host.

## Reproducing the analysis

1. Install Python 3.11 or a compatible version.
2. Create and activate a virtual environment.
3. Install the dependencies listed in `requirements.txt`.
4. Obtain the original lab capture from its source, subject to its
   license and redistribution terms.
5. Place the capture at the path expected by the notebook:
   `data/worm/extraido/pacotes_worm.pcapng`.
6. Open `Worm_445_Analise.ipynb` in Jupyter or VS Code and run the cells.

The PCAP file is not included in this repository. The notebook's
relative input path must be preserved when reproducing the analysis.

## Data source and credits

The lab and original packet capture were obtained from:

Carlos Silva — https://github.com/carlossilva-cybersec/hackone_pos

Scapy, Pandas, and Scikit-learn are used as analysis tools.

This repository documents independent educational analysis. It is
not an official publication by the original lab author. Please check
the source repository's license and applicable permissions before
redistributing its materials or packet capture.

## Limitations

- Results depend on the available packet capture and its visibility.
- Anomaly detection identifies unusual behavior; it does not establish
  malicious intent on its own.
- Scanning activity does not prove successful compromise.
- IP addresses are interpreted in the context of this capture.

## Disclaimer

For educational and defensive cybersecurity purposes only.
