# Dataset IoT HTTP/2 Rapid Reset - Dataset e guia de reprodução - MENTORED TESTBED

> [!IMPORTANT]
> A seguir são descritos os passos necessários para reproduzir e analisar os experimentos realizados no MENTORED Testbed a partir dos arquivos de definição de experimento. [Para baixar o dataset gerado utilize este link](https://drive.google.com/drive/folders/1N1-0S1FixL5t0iuXHI8HgYrQtpkSjhXW?usp=sharing)

> [!NOTE]
> É recomendado a leitura dos [tutoriais](https://portal.mentored.ccsc-research.org/tutorial/pt/) do projeto primeiramente. Além disso recomenda-se como ferramentas o Wireshark, um editor de texto como VSCode, assim como um leitor de arquivos compactados como 7zip ou Peazip.


## Objetivos

1. Familiarizar o leitor com o MENTORED _testbed_

2. Executar experimentos pré-existentes

3. Examinar dados coletados para avaliar a efetividade dos ataques performados

### HTTP/2 RAPID RESET (Estimativa de 10min)

Esse cenário (`CenarioRapidReset.yaml`) tem como objetivo demonstrar a execução de um ataque recente no MENTORED _testbed_. Em específico, o HTTP/2 RAPID RESET, o qual é melhor explicado [neste blog post da Cloudflare](https://blog.cloudflare.com/technical-breakdown-http2-rapid-reset-ddos-attack/).


#### Entidades

1) Servidor Web Apache: servindo uma página Web estática via HTTP 1.1

2) Proxy Reverso Nginx: um servidor Nginx agindo como proxy reverso para o servidor supracitado, servindo conexões HTTP/2 via HTTPS para clientes

2) Clientes Web: um cliente HTTP/2 escrito em Python 3 executando requisições periódicas ao servidor web, com criptografia TLS (HTTPS)

3) Atacante: um node actor com quatro containers, cada um realizando ataques de bruteforce SSH contra um nó especifico. O segundo estágio deste ataque é a execução do ataque HTTP/2 RAPID RESET, em especifico [esta implementação de tal ataque](https://github.com/secengjeff/rapidresetclient).

4) Nós vulneráveis: node actors (entidades) que simulam um dispositivo IoT qualquer com a porta 22/TCP (SSH) exposta para a internet, com uma senha simples

![alt text](img/Diagramas-Cenários-Distribuição-C3.png)


#### Fluxo / Timeline

* Duração 300 segundos
* [0-59s] Clientes legítimos realizam requisições normalmente contra o servidor Web

* [60-240s] Atacante inicia tentativas de bruteforce SSH contra os nós vulneráveis. Tais nós quando comprometidos iniciam o ataque previamente descrito por 180 segundos.

* [241-300s] Apenas clientes legítimos e o servidor Web estão ativos

![alt text](img/Diagramas-Cenários-Timeline-C3.png)

#### Dados coletados por entidade

* Todos:
  
  * Logs do script de inicialização (arquivo `experiment_logs_<xyz>.tar`)
  
  * IPs de cada nó (`MENTORED_IP_LIST.yaml`)
  
  * Tempo de inicialização do experimento (`MENTORED_READY.txt`)

* Servidor Web Apache:
  
  * Captura de tráfego de rede (`/app/results/packets.pcapng`, arquivo assim como arquivo `hosts` na mesma pasta)
  
  * Logs do Apache 2 (`/app/results/access.log` e `error.log` na mesma pasta)

* Proxy Reverso Nginx:

  * Logs de acesso (`/var/log/nginx/access.log` e `error.log` na mesma pasta)
  
  * Captura de tráfego de rede (`/app/results/packets.pcapng`)

* Cliente:

  * Registro (timestamp) do início das requisições(`/app/results/MENTORED_REGISTRY.yaml`)
  
  * CSV contando a latência de cada requisição ou um erro demarcando falha na conexão (`/app/results/client_delay.csv`)

* Atacante:
  
  * Registro (timestamp) do início e fim do ataque SSH (`/app/results/MENTORED_REGISTRY.yaml`)

* Nó IoT Vulnerável:

  * Registros do inicio e fim do ataque rapidreset (`/app/results/MENTORED_REGISTRY.yaml`)

  * Logs de acesso do servidor SSH (`/app/results/sshd.log`)

#### Executando o cenário

Navegue pelo portal e criar uma definição de experimento, desta vez utilizando o `CenarioRapidReset.yaml` como base.

Durante o experimento, isto é, após o final do warmup, analise o conteúdo dos arquivos listados na seção Dados Coletados por Entidade. Use comandos como `tail -f /<caminho>` para acompanhar o conteúdo dos arquivos em tempo real ou monitore os processos ativos de uma entidade, como nós SSH vulneráveis, utilizando `watch ps aux`. Além disso, procure a ocorrência das requisições dos nós realizando o ataque nos registros de acesso do proxy reverso nginx, como exposto abaixo:

```
10.194.6.11 - - [09/Dec/2024:21:13:43 +0000] "GET / HTTP/2.0" 499 0 "-" "-"
10.194.8.11 - - [09/Dec/2024:21:13:43 +0000] "GET / HTTP/2.0" 499 0 "-" "-"
10.194.12.2 - - [09/Dec/2024:21:13:43 +0000] "GET / HTTP/2.0" 499 0 "-" "-"
10.194.13.2 - - [09/Dec/2024:21:13:43 +0000] "GET / HTTP/2.0" 499 0 "-" "-"
10.194.6.11 - - [09/Dec/2024:21:13:43 +0000] "GET / HTTP/2.0" 499 0 "-" "-"
10.194.8.11 - - [09/Dec/2024:21:13:43 +0000] "GET / HTTP/2.0" 499 0 "-" "-"
10.194.12.2 - - [09/Dec/2024:21:13:43 +0000] "GET / HTTP/2.0" 499 0 "-" "-"
```

Alguns outros comandos de interesse são listados abaixo podem ser executados em um cliente por exemplo:

```bash
cat /MENTORED_READY # timestmap inicio do experimento
cat /etc/hosts # registros DNS fixos realizados automaticamente
ping -c3 server-http # teste de conectividade ao servidor via ICMP
curl -k -vvv https://server-http # tentativa de requisição da página web na raiz
curl -k -v https://server-http/home?min_words=1&max_words=7478 # solicitação de uma página com conteúdo de tamanho variável
```

#### Análise dos resultados

Utilize as ferramentas de análise dados do cliente e servidor para extrair conhecimento sobre a efetividade deste ataque.

1) Baixe o arquivo `.tar.gz` e copie o mesmo para esta pasta do roteiro clonada anteriormente (adaptando o comando abaixo)

```bash
MY_ATTACK_START=60
MY_ATTACK_END=240
MY_EXP_FILE=experiment_XXXX.tar.gz
cp ~/Downloads/$MY_EXP_FILE .
```

2) Execute o script de análise

> [!NOTE]
> Abaixo um exemplo utilizando a imagem Docker pré-construída


```bash
sudo docker run --rm -it \
    -v .:/app ghcr.io/khalilsantana/dataset-mentored-iot-2024 \
    python3 /app/scripts/clients-analysis/client_metrics.py \
    ../../$MY_EXP_FILE -a $MY_ATTACK_START -p $MY_ATTACK_END
```

3) Observe os resultados, a exemplo abaixo:

```
INFO - Average time for client response (Before 60 seconds)    : 0.062 - 0 errors
INFO - Average time for client response (60 - 240 seconds)      : 0.127 - 5 errors
INFO - Average time for client response (After 240 seconds)     : 0.094 - 1 errors
```

Isto é, houveram 5 erros de requisição durante o ataque, com a latencia média aproximadamente o dobro periodo pré-ataque (127ms vs 62ms). Desta maneira comprovamos que o ataque HTTP/2 RAPID RESET foi efeitivo neste cenário! Contudo tal ataque tem uma efetividade variada, multiplas execuções podem resultar em resultados diferentes.

**Análise de dados do servidor**

1) Dentro da pasta do repostório do roteiro, certifique-se que o arquivo do experimento está presente e estas variáveis de ambiente estão configuradas

```bash
MY_EXP_DURATION=300
MY_EXP_FILE=experiment_XXXX.tar.gz
cp ~/Downloads/$MY_EXP_FILE .
```

2) Execute o script de análise de dados de servidor

> [!NOTE] 
> Abaixo um exemplo utilizando a imagem Docker pré-construída


```bash
sudo docker run --rm -it \
    -v .:/app ghcr.io/khalilsantana/dataset-mentored-iot-2024 \
    /app/scripts/server-analysis/experiment_analyzer.sh ../../$MY_EXP_FILE $MY_EXP_DURATION
```

3) Observe os resultados, em `scripts/server-analysis/output_${TIMESTAMP}.png`

![alt text](img/C3-RapidReset.png)

# Créditos extras

<a href="https://www.flaticon.com/free-icons/cyber-attack" title="cyber attack icons">Cyber attack icons created by Freepik - Flaticon</a>