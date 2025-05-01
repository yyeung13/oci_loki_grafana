# Loki Configuration on OCI with Integration to Grafana
This repository demonstrates how to configure Loki on OCI to receive logs via Promtail and configure Loki as a Data Source for Grafana

<b> Pre-requisite</b><br>

The following guide assumes you already provision the necessary compute(s) for Loki, Grafana and the compute where logs are generating on OCI.<br>
<br>
<b>Step 1: Installation and Configuration of Loki</b><br>
To install Loki on an OCI Compute instance (such as a standard VM, not Kubernetes), using the default configuration, follow these steps:<br>
<br>
1. Prepare Your OCI Compute Instance<br>
	•	Deploy a Linux VM (Oracle Linux, Ubuntu, or CentOS recommended).<br>
	•	Ensure you have root or sudo access.<br>
	•	Install basic utilities: `curl`, `wget`, and `tar`.<br>
 <br>
2. Download Loki Binary<br>
	•	Go to the official Loki releases page or use the following commands to download the latest Loki binary:<br>
<pre>
# Example for Linux AMD64 (x86_64)
wget https://github.com/grafana/loki/releases/latest/download/loki-linux-amd64.zip
unzip loki-linux-amd64.zip
chmod +x loki-linux-amd64
sudo mv loki-linux-amd64 /usr/local/bin/loki
</pre>
<br>
3. Create a Default Configuration File<br>
	•	Loki requires a configuration file, usually named `loki.yaml`. A sample is provided in this repository.<br>
	•	You can generate a default config or download the sample from Grafana:<br>
 <br>
 <pre>
   wget https://raw.githubusercontent.com/grafana/loki/main/cmd/loki/loki-local-config.yaml -O loki.yaml
 </pre>
 <br>
 	•	For a basic test, this default config will store logs locally on disk. You can edit `loki.yaml` to adjust storage or network settings later.<br>
  <br>
4. Start Loki<br>
	•	Run Loki with the configuration file:<br>
 <pre>
   loki -config.file=loki.yaml
 </pre>
<br>
	•	By default, Loki listens on port 3100. You can check logs in your terminal to verify it is running.<br>
 <br>
 5. (Optional) Configure as a Service<br>
	•	To keep Loki running in the background, create a simple systemd service:<br>
 <pre>
   sudo nano /etc/systemd/system/loki.service
 </pre>
<br>
Paste the following:<br>
<pre>
  [Unit]
Description=Grafana Loki Service
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/loki -config.file=/path/to/loki.yaml
Restart=on-failure

[Install]
WantedBy=multi-user.target

</pre>
 <br>
 	•	Reload systemd and start Loki:<br>
  <pre>
    sudo systemctl daemon-reload
    sudo systemctl enable --now loki
  </pre>
  <br>
  6. Loki is Ready<br>
	•	Loki is now running on your OCI Compute instance with default configuration.<br>
	•	Point your log shippers (like Promtail or Grafana Agent) to `http://<your-instance-ip>:3100`.<br>
    <br>
Note:<br>
	•	For production, configure persistent storage and consider integrating with OCI Object Storage using the S3-compatible API.<br>
	•	For advanced configuration, see the official Loki documentation.<br>
 <br>
<b>Step 2: Installation and Configuration of Promtail in the same VM hosting target log files</b><br>
<br>
Promtail is Grafana Loki’s agent for shipping logs from local files to a Loki server. Here’s how to install and configure Promtail on an Oracle Cloud Infrastructure (OCI) compute instance using the default binary method.<br>
1. Download and Install Promtail<br>
	•	Download the latest Promtail binary from the official Loki releases page or use:  <br>
<pre>
  wget https://github.com/grafana/loki/releases/latest/download/promtail-linux-amd64.zip
  unzip promtail-linux-amd64.zip
  chmod +x promtail-linux-amd64
  sudo mv promtail-linux-amd64 /usr/local/bin/promtail
</pre>
<br>
2. Create a Promtail Configuration File<br>
	•	Create a file named `promtail-config.yaml` in your home directory or `/etc/promtail/`. A sample config is provided in this repository.<br>
	•	Here’s a minimal example configuration (edit the Loki URL as needed):<br>
<br>
<pre>
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /home/opc/positions.yaml

clients:
  - url: http://\<loki-server-ip\>:3100/loki/api/v1/push

scrape_configs:
  - job_name: system
    static_configs:
      - targets:
          - localhost
        labels:
          job: varlogs
          __path__: /home/opc/*.log

</pre>
<br>
•	Replace `&lt;loki-server-ip&gt;` with the IP address or DNS name of your Loki server.<br>
<br>
3. Start Promtail<br>
	•	Run Promtail with your configuration file:<br>
<pre>
  promtail -config.file=promtail-config.yaml
</pre>
<br>
Promtail will now begin shipping logs from `/var/log/*.log` to your Loki instance.<br>
  <br>
4. (Optional) Run Promtail as a Service<br>
	•	To keep Promtail running in the background, create a systemd service:<br>
 <pre>
   sudo nano /etc/systemd/system/promtail.service
 </pre>
 <br>
 Paste the following:<br>
 <pre>[Unit]
  Description=Grafana Promtail Service
  After=network.target
  
  [Service]
  Type=simple
  ExecStart=/usr/local/bin/promtail -config.file=/path/to/promtail-config.yaml
  Restart=on-failure
  
  [Install]
  WantedBy=multi-user.target

 </pre>
<br>
	•	Reload and start the service:<br>
 <pre>
  sudo systemctl daemon-reload
  sudo systemctl enable --now promtail

 </pre>
 <br>
5. Verify Operation<br>
	•	Check logs to ensure Promtail is running and sending data:<br>
 <pre>
   journalctl -u promtail -f
 </pre>
 <br>
 Notes<br>
	•	Promtail is in LTS and will reach EOL on March 2, 2026; consider migration plans for the future.<br>
	•	You can customize which logs are shipped by editing the `scrape_configs` section in your config file.<br>
	•	For advanced configuration options, refer to the Promtail documentation.<br>
 <br>
 
<b>Step 3: Installation and Configuration of Grafana</b><br>
<br>
These steps guide you through installing Grafana on an OCI compute instance (VM), connecting it to a Loki server, and visualizing logs.<br>
<br>
1. Provision and Prepare Your OCI Compute Instance<br>
	•	Launch a Linux VM in OCI with internet access.<br>
	•	Update your system and install dependencies:<br>
 <pre>
  sudo yum update -y   # For Oracle Linux/CentOS
  sudo yum install -y wget

 </pre>
 <br>
 2. Install Grafana<br>
 	•	Prepare a file grafana.repo under /etc/yum.repos.d (repo file in this repository><br>
	•	Download and install Grafana (for Oracle Linux/CentOS):<br>
  <pre>
sudo yum install -y grafana
</pre>
<br>
	•	Start and enable Grafana:<br>
 <pre>
sudo systemctl start grafana-server
sudo systemctl enable grafana-server

 </pre>
<br>
3. Access Grafana Web UI<br>
	•	Open port 3000 in your OCI security list.<br>
	•	In your browser, go to: `http://<your-oci-public-ip>:3000`<br>
	•	Login with default credentials:<br>
	•	Username: `admin`<br>
	•	Password: `admin` (you will be prompted to change this)<br>
Note: Disable OCI firewall if the port is not accessible<br>
<br>
4. Add Loki as a Data Source in Grafana<br>
	•	In the Grafana UI, click the gear icon (Configuration) on the left.<br>
	•	Select Data Sources > Add data source.<br>
	•	Search for and select Loki as the data source type.<br>
	•	In the HTTP URL field, enter your Loki server’s address, e.g.:<br>
	•	`http://<loki-server-ip>:3100`<br>
	•	Click Save & Test to verify the connection.<br>
<br>
5. Explore and Visualize Logs<br>
	•	Click Explore in the left menu.<br>
	•	In the Query field, select your Loki data source.<br>
	•	Use label filters (e.g., `{job="varlogs"}`) to view logs shipped by Promtail or other agents.<br>
	•	You can create dashboards and panels to visualize log data as needed.<br>
<br>
6. (Optional) Secure and Harden Your Setup<br>
	•	Change default passwords and restrict access to Grafana.<br>
	•	Consider enabling HTTPS for the Grafana web interface.<br>
	•	Regularly update Grafana and plugins.<br>
 <br>
 There you go, you have configured Loki to receive logs through Promtail, and have setup Grafana to use Loki as a Data Source for logs visualization.<br>
