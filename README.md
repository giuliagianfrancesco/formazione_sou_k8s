# Jenkins Controller/Agent su Vagrant con provisioning Ansible + Docker + Kubernetes



Provisioning automatizzato di un'infrastruttura Jenkins CI/CD (Controller + Agent inbound), con pipeline di build/push Docker e deploy su Kubernetes via Helm.



## Architettura



- **Vagrant + VirtualBox**: VM Rocky Linux 9

- **Ansible** (`ansible_local`): provisioning Docker, rete, volumi, container

- **Docker**: esecuzione di Jenkins Controller e Agent come container isolati

- **Jenkins**: Controller/Agent in modalità **inbound**

- **Ansible Vault**: gestione cifrata del secret dell'agent

- **Minikube**: cluster Kubernetes target del deploy

- **Helm**: gestione del deployment applicativo







## Setup



```bash

vagrant up

```



Eseguendo il playbook Ansible:

1. Installa Docker

2. Crea rete `jenkins_network` (bridge, DNS per risoluzione per nome tra container)

3. Crea volume `jenkins_home` (persistenza dati)

4. Avvia `jenkins-controller` (`jenkins/jenkins:lts`)

5. Avvia `jenkins-agent` (`jenkins/inbound-agent`), con `JENKINS_SECRET` letto da un file cifrato con Ansible Vault



## Collegamento Agent → Controller



Modalità **inbound** (agent si connette al controller): Il secret del nodo va generato da UI (`Manage Jenkins → Nodes`) al primo avvio e poi passato via `--vault-password-file` per ogni esecuzione del playbook:



```bash

ansible-playbook -i inventory/hosts.ini playbook_provision.yml --vault-password-file .vault_pass.txt

```



Verifica connessione:

```bash

docker logs jenkins-agent

```



## Pipeline



Job Jenkins di tipo **Pipeline script from SCM**, che legge il `Jenkinsfile` dal repository.

Stage principali:



1. **Checkout** — clona il repository

2. **Tag version** — determina il tag immagine da branch/tag Git

3. **Build** — build e push immagine Docker su Docker Hub

4. **Deploy su cluster con Helm** — `helm upgrade --install` con `--set image.tag=...` per coerenza tra immagine buildata e deployata

5. **Export Deployment** — esporta stato del deployment via `kubectl`

## Connessione verso il cluster Kubernetes



Il cluster Minikube espone l'API server solo su `127.0.0.1` del Mac quindi non raggiungibile dalla VM. La soluzione è usare `kubectl proxy` sul Mac, che espone l'API su tutte le interfacce in HTTP, raggiunto dalla VM tramite indirizzo NAT standard di VirtualBox (`10.0.2.2`).



```bash

kubectl proxy --address='0.0.0.0' --accept-hosts='^.*$' --port=8001 &

```
