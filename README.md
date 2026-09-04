# Elastic Defend EDR Home Lab

## Objectives

This lab simulates deploying Elastic's EDR product (Elastic Defend) on a self-managed Elastic Stack, then validating that detection and prevention actually work end-to-end. The setup includes:

- **Ubuntu Server VM (192.168.1.158)**: hosts Elasticsearch, Kibana, Fleet Server, and the monitored Elastic Agent, all running on a single machine via VirtualBox with bridged networking
- **Elastic Stack (Docker)**: deployed using Elastic's official `start-local` script
- **Fleet Server**: manages agent enrollment and policy distribution
- **Elastic Agent + Elastic Defend integration**: configured at the Complete EDR tier — full malware/ransomware/behavioral prevention plus process, file, and network telemetry

Unlike traditional antivirus, which mostly scans files against known malware signatures, EDR continuously monitors endpoint activity to detect and respond to threats in real time, including ones that don't match a known signature. This lab proves that capability using an industry-standard safe test file (EICAR).

## Setting Up Elasticsearch, Kibana, and Fleet Server

Deployed Elasticsearch and Kibana via Elastic's `start-local` Docker script, running on the Ubuntu VM.

<img width="1279" height="698" alt="kibana-home" src="https://github.com/user-attachments/assets/62248050-ba4b-45ee-99be-badf1c808baf" />




Kibana was reachable at `http://192.168.1.158:5601` after fixing a Docker networking issue — the default `docker-compose.yml` bound Kibana to `127.0.0.1:5601` only, which is unreachable from outside the container. Rebound to `0.0.0.0:5601`.

Started the Fleet Server setup from Fleet → Agents → Add Fleet Server:

<img width="514" height="658" alt="Screenshot 2026-09-03 225722" src="https://github.com/user-attachments/assets/a4faf1e6-7b40-4974-a096-a6aec8c00a7b" />



## Fixing a Fleet Server Authentication Failure

The Kibana "Quick Start" install command failed twice with:

```
Error: fleet-server failed: timed out waiting for Fleet Server to start after 2m0s
```

Diagnosed the root cause: `curl http://localhost:9200` returned a `security_exception` — the Quick Start command didn't include Elasticsearch authentication credentials, so Fleet Server couldn't connect.

<img width="1148" height="1043" alt="image" src="https://github.com/user-attachments/assets/c6bffcf5-6729-4a07-90b1-46ad246d9f9d" />


**Fix:** switched to the "Advanced" setup path, which generates a proper **service token** — a scoped Elasticsearch credential — and bakes it into the install command via `--fleet-server-service-token`.

<img width="1148" height="1043" alt="image" src="https://github.com/user-attachments/assets/dea667a2-de76-4dd4-a1df-d298c0fc99ed" />


Fleet Server installed and enrolled successfully:

<img width="1277" height="698" alt="fleet-server-healthy" src="https://github.com/user-attachments/assets/a7d7b203-f51d-42e7-967e-052967ff0161" />

## Deploying Elastic Defend

Added the Elastic Defend integration to the same agent policy already running Fleet Server, configured at the **Complete EDR** tier — the highest of four available levels (Data Collection → NGAV → Essential EDR → Complete EDR).

![elastic defend configuration](images/defend-config.png)

Confirmed the endpoint as **Healthy** under Security → Assets → Endpoints. (Note: Kibana's Security navigation is hidden by default in `start-local` deployments, since the space defaults to an Elasticsearch-only "Solution View" — found it via Stack Management → Spaces.)

<img width="1280" height="695" alt="endpoint-healthy" src="https://github.com/user-attachments/assets/07c62c9f-653d-4508-951a-e0d98147ce9f" />



## Validating Detection: EICAR Malware Test

Used the [EICAR test file](https://www.eicar.org/) — an industry-standard, harmless string that every AV/EDR engine is designed to recognize as a test signature — to confirm the pipeline actually detects and responds to threats.

```bash
curl -o ~/eicar.txt https://secure.eicar.org/eicar.com.txt
```

The download completed (68 bytes transferred), but the file was gone immediately afterward:

```
$ ls -la ~/eicar.txt
ls: cannot access '/home/vboxuser/eicar.txt': No such file or directory
```

Elastic Defend's malware prevention engine caught and removed the file in real time.

<img width="1148" height="1043" alt="image" src="https://github.com/user-attachments/assets/fe82858f-20b3-4e3f-99f6-3ef3b6ce7b6c" />


Confirmed in Kibana's Alerts dashboard:

| Field | Value |
|---|---|
| Rule | Malware Prevention Alert |
| Severity | High |
| Risk Score | 73 |
| Host | ubuntu-elastic |
| Reason | malware, intrusion_detection, file event with process context |

<img width="1280" height="764" alt="eicar-alert" src="https://github.com/user-attachments/assets/1dd2b63c-81a5-4b6a-9707-cf4e3c4815bf" />


## Conclusion

This project demonstrates deploying a self-managed Elastic Stack (Elasticsearch, Kibana, Fleet Server) and Elastic Defend EDR from scratch, diagnosing and resolving a real authentication failure between Fleet Server and Elasticsearch, configuring an EDR policy at production-realistic settings, and validating detection/prevention capability end-to-end using the EICAR test standard. This project is intentionally kept separate from the Wazuh SIEM + Honeypot lab — Elastic and Wazuh are separate, competing SIEM/EDR ecosystems with no native integration, and running both independently reflects how many real environments juggle multiple, non-unified security tools.
