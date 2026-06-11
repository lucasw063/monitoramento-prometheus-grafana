# monitoramento-prometheus-grafana
Stack de monitoramento com Prometheus + Grafana em VM na Oracle Cloud.

Monitoramento com Prometheus e Grafana
> Documentação técnica de instalação, configuração e integração do stack de monitoramento Prometheus + Grafana em instância Ubuntu 22.04 na Oracle Cloud Infrastructure (OCI).
---
1. Introdução à Ferramenta
Prometheus
O Prometheus é um sistema open-source de monitoramento e alerta criado pela SoundCloud em 2012 e atualmente mantido pela Cloud Native Computing Foundation (CNCF). Ele funciona como um banco de dados de séries temporais (TSDB), coletando métricas de alvos configurados em intervalos regulares via protocolo HTTP.
Suas principais características são:
Modelo de dados multidimensional com métricas identificadas por nome e pares chave-valor (labels)
Linguagem de consulta própria chamada PromQL
Coleta de métricas por pull (o Prometheus busca os dados nos exporters)
Suporte a service discovery e configuração estática de alvos
Sistema de alertas integrado via Alertmanager
Grafana
O Grafana é uma plataforma open-source de visualização e observabilidade. Ele se conecta a diversas fontes de dados — incluindo o Prometheus — e permite a criação de dashboards interativos, painéis de métricas em tempo real, alertas visuais e análise histórica de dados.
Integração Prometheus + Grafana
A integração entre as duas ferramentas segue o seguinte fluxo:
```
[ Sistema Operacional ]
        ↓
[ Node Exporter :9100 ]   ← expõe métricas do SO via HTTP
        ↓
[ Prometheus :9090 ]      ← coleta (scrape) as métricas a cada 15s
        ↓
[ Grafana :3000 ]         ← consulta o Prometheus via PromQL e exibe dashboards
        ↓
[ Nginx :80 ]             ← proxy reverso expondo o Grafana em /grafana
        ↓
[ Usuário via browser ]
```
---
2. Guia de Instalação e Configuração Básica
Pré-requisitos
Ubuntu 22.04 LTS
Usuário com permissões `sudo`
Portas 9090, 9100 e 3000 liberadas no firewall
2.1 Node Exporter
O Node Exporter é responsável por coletar métricas do sistema operacional (CPU, memória, disco, rede) e expô-las em `/metrics` na porta `9100`.
```bash
# Download e instalação
cd ~
wget https://github.com/prometheus/node_exporter/releases/download/v1.8.1/node_exporter-1.8.1.linux-amd64.tar.gz
tar xvf node_exporter-1.8.1.linux-amd64.tar.gz
sudo mv node_exporter-1.8.1.linux-amd64/node_exporter /usr/local/bin/

# Criação do serviço systemd
sudo tee /etc/systemd/system/node_exporter.service << EOF
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=ubuntu
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
EOF

# Ativação e inicialização
sudo systemctl daemon-reload
sudo systemctl enable node_exporter
sudo systemctl start node_exporter
sudo systemctl status node_exporter
```
2.2 Prometheus
```bash
# Download e instalação
cd ~
wget https://github.com/prometheus/prometheus/releases/download/v2.52.0/prometheus-2.52.0.linux-amd64.tar.gz
tar xvf prometheus-2.52.0.linux-amd64.tar.gz
sudo mv prometheus-2.52.0.linux-amd64/prometheus /usr/local/bin/
sudo mv prometheus-2.52.0.linux-amd64/promtool /usr/local/bin/
sudo mkdir -p /etc/prometheus /var/lib/prometheus
sudo mv prometheus-2.52.0.linux-amd64/consoles /etc/prometheus/
sudo mv prometheus-2.52.0.linux-amd64/console_libraries /etc/prometheus/

# Correção de permissões (necessário para evitar erro de mmap)
sudo chown -R ubuntu:ubuntu /var/lib/prometheus
```
Arquivo de configuração `/etc/prometheus/prometheus.yml`:
```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node_exporter'
    static_configs:
      - targets: ['localhost:9100']
```
```bash
# Criação do serviço systemd
sudo tee /etc/systemd/system/prometheus.service << EOF
[Unit]
Description=Prometheus
After=network.target

[Service]
User=ubuntu
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable prometheus
sudo systemctl start prometheus
sudo systemctl status prometheus
```
2.3 Grafana
```bash
# Instalação via repositório oficial
sudo apt-get install -y apt-transport-https software-properties-common
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
echo "deb https://packages.grafana.com/oss/deb stable main" | sudo tee /etc/apt/sources.list.d/grafana.list
sudo apt-get update
sudo apt-get install -y grafana
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
```
Configuração do subpath em `/etc/grafana/grafana.ini`:
```bash
sudo sed -i 's/^;domain =.*/domain = lwadigital.com.br/' /etc/grafana/grafana.ini
sudo sed -i 's/^;root_url =.*/root_url = http:\/\/lwadigital.com.br\/grafana\//' /etc/grafana/grafana.ini
sudo sed -i 's/^;serve_from_sub_path =.*/serve_from_sub_path = true/' /etc/grafana/grafana.ini
sudo systemctl restart grafana-server
```
2.4 Nginx como Proxy Reverso
Configuração em `/etc/nginx/sites-available/default`:
```nginx
server {
    listen 80;
    server_name lwadigital.com.br www.lwadigital.com.br;

    location / {
        root /var/www/html;
        index index.html;
        try_files $uri $uri/ =404;
    }

    location /grafana/ {
        proxy_pass http://localhost:3000/grafana/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```
```bash
sudo nginx -t && sudo systemctl restart nginx
```
2.5 Liberação de Portas
```bash
sudo iptables -I INPUT -p tcp --dport 3000 -j ACCEPT
sudo iptables -I INPUT -p tcp --dport 9090 -j ACCEPT
sudo iptables -I INPUT -p tcp --dport 9100 -j ACCEPT
sudo netfilter-persistent save
```
> Além do firewall interno, as portas 3000, 9090 e 9100 também precisam ser liberadas no **Security List da VCN** no console da Oracle Cloud.
---
3. Execução Local e Testes
Verificar status dos serviços
```bash
sudo systemctl status node_exporter
sudo systemctl status prometheus
sudo systemctl status grafana-server
```
Validar endpoints
```bash
# Node Exporter expondo métricas
curl http://localhost:9100/metrics | head -20

# Prometheus health check
curl http://localhost:9090/-/healthy

# Grafana respondendo
curl -v http://localhost:3000 2>&1 | grep "HTTP/"
```
Acessos via browser
Serviço	URL
Site principal	`http://lwadigital.com.br`
Grafana	`http://lwadigital.com.br/grafana`
Prometheus (direto)	`http://IP_DA_VM:9090`
Node Exporter (métricas)	`http://IP_DA_VM:9100/metrics`
Credenciais padrão do Grafana
Usuário: `admin`
Senha: `admin` (alterada no primeiro acesso)
Importar dashboard Node Exporter Full
No Grafana: Dashboards → New → Import
ID do dashboard: `1860`
Selecionar o datasource Prometheus criado
Clicar em Import
Verificar logs
```bash
# Logs do Prometheus
sudo journalctl -u prometheus -f

# Logs do Grafana
sudo journalctl -u grafana-server -f

# Logs do Node Exporter
sudo journalctl -u node_exporter -f
```
---
4. Integração com o Serviço
Serviço integrado: Node Exporter (métricas do sistema host)
O Node Exporter atua como o exporter responsável por coletar métricas diretamente do sistema operacional da VM Ubuntu 22.04 hospedada na Oracle Cloud Infrastructure.
Como a comunicação é feita:
O Node Exporter expõe um endpoint HTTP em `localhost:9100/metrics` no formato de texto Prometheus (exposition format). O Prometheus, configurado com o job `node_exporter`, realiza requisições GET a esse endpoint a cada 15 segundos (conforme o `scrape_interval` definido no `prometheus.yml`) e armazena as séries temporais localmente em `/var/lib/prometheus`.
O Grafana então consulta o Prometheus via API HTTP usando PromQL para renderizar os dashboards.
Principais métricas coletadas:
Métrica	Descrição
`node_cpu_seconds_total`	Tempo de CPU por modo (user, system, idle)
`node_memory_MemAvailable_bytes`	Memória disponível em bytes
`node_filesystem_avail_bytes`	Espaço disponível em disco
`node_network_receive_bytes_total`	Bytes recebidos pela interface de rede
`node_network_transmit_bytes_total`	Bytes transmitidos pela interface de rede
`node_load1`	Carga média do sistema em 1 minuto
---
5. Dificuldades Enfrentadas
5.1 Erro de permissão no diretório de dados do Prometheus
Sintoma: O serviço do Prometheus falhava ao iniciar com o seguinte erro:
```
panic: Unable to create mmap-ed active query log
```
Causa: O diretório `/var/lib/prometheus` foi criado com permissões de root, mas o serviço estava configurado para rodar com o usuário `ubuntu`.
Solução:
```bash
sudo chown -R ubuntu:ubuntu /var/lib/prometheus
sudo systemctl restart prometheus
```
---
5.2 Loop de redirecionamento (ERR_TOO_MANY_REDIRECTS) no Grafana via Nginx
Sintoma: Ao acessar `lwadigital.com.br/grafana`, o browser retornava `ERR_TOO_MANY_REDIRECTS`.
Causa: O `proxy_pass` no Nginx apontava para `http://localhost:3000/` (sem o subpath), fazendo com que o Grafana respondesse com um redirect para `/grafana/`, que era novamente interceptado pelo Nginx e reenviado para `localhost:3000/`, criando um loop infinito.
Solução: Corrigir o `proxy_pass` para incluir o subpath `/grafana/`:
```nginx
location /grafana/ {
    proxy_pass http://localhost:3000/grafana/;
    ...
}
```
---
5.3 Grafana não aceitando `localhost` como datasource
Sintoma: Ao configurar o Prometheus como datasource no Grafana com a URL `http://localhost:9090`, retornava erro de conexão recusada.
Causa: O Prometheus estava parado devido ao erro de permissão descrito no item 5.1.
Solução: Corrigir as permissões e reiniciar o Prometheus antes de testar o datasource.
---
5.4 Erro 521 da Cloudflare (Web Server is Down)
Sintoma: O domínio `lwadigital.com.br` retornava erro 521 da Cloudflare.
Causa: A Cloudflare estava tentando se conectar ao servidor via HTTPS (porta 443), mas o Nginx só estava configurado para escutar na porta 80.
Solução: Alterar o modo SSL da Cloudflare de Full para Flexible, fazendo com que a Cloudflare se comunique com o servidor via HTTP enquanto mantém HTTPS para o visitante.
---
5.5 Configurações do grafana.ini ignoradas (linhas comentadas)
Sintoma: Após editar o `/etc/grafana/grafana.ini` com `nano`, as configurações de `domain`, `root_url` e `serve_from_sub_path` não eram aplicadas.
Causa: As linhas no arquivo estavam comentadas com `;` e a edição manual não removeu o caractere de comentário corretamente.
Solução: Usar `sed` para substituir diretamente as linhas comentadas:
```bash
sudo sed -i 's/^;domain =.*/domain = lwadigital.com.br/' /etc/grafana/grafana.ini
sudo sed -i 's/^;root_url =.*/root_url = http:\/\/lwadigital.com.br\/grafana\//' /etc/grafana/grafana.ini
sudo sed -i 's/^;serve_from_sub_path =.*/serve_from_sub_path = true/' /etc/grafana/grafana.ini
```
---
6. Conclusão
O stack Prometheus + Grafana é ideal para projetos que demandam observabilidade contínua de infraestrutura e aplicações, especialmente em ambientes cloud, servidores Linux e arquiteturas de microsserviços. A combinação oferece coleta eficiente de métricas via pull, armazenamento local de séries temporais, dashboards altamente customizáveis e um ecossistema rico de exporters para praticamente qualquer tecnologia.
Recomendamos essa stack para projetos de médio a grande porte que precisam monitorar múltiplos serviços simultaneamente, equipes DevOps que buscam uma solução open-source robusta sem custo de licenciamento, e qualquer projeto hospedado em nuvem que precise de visibilidade sobre o consumo de recursos em tempo real. No contexto deste projeto, a integração com a Oracle Cloud Free Tier demonstrou que é possível ter um ambiente de monitoramento completo e profissional sem custo algum de infraestrutura.
