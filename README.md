# promestheusLab
Certification and Authentication between Prometheus server and targets
** on node_exporter server
1) Encryption
   The first thing that we have to do is create certificates.
   I’m using open SSL on node_exporter to generate self-signed certificates.
   
         $ sudo openssl req -new -newkey rsa:2048 -days 365 -nodes -x509 -keyout node_exporter.key -out node_exporter.crt -subj 
        "/C=US/ST=California/L=Oakland/O=MyOrg/CN=localhost" -addext "subjectAltName = DNS:localhost"

and Create a folder in node_exporter in /etc

         $ sudo mkdir /etc/node_exporter
         
then copy key and certificed genereted under the /etc/node_exporter

        $ cp node_exporter.key node_exporter.crt /etc/node_exporter/
        
then Create a config.ymlfile in the same directory

      tls_server_config:
        cert_file: node_exporter.crt
        key_file: node_exporter.key

Update the permission to the folder for the user node_exporter

      $ sudo chown -R node_exporter:node_exporter /etc/node_exporter

Update the systemd service of node_exporter with TLS config 

     $ sudo vi /etc/systemd/system/node_exporter.service

      [Unit]
      Description=Node Exporter
      Wants=network-online.target
      After=network-online.target
      [Service]
      User=node_exporter
      Group=node_exporter
      Type=simple
      ExecStart=/usr/local/bin/node_exporter --web.config.file="/etc/node_exporter/config.yml"
      [Install]
      WantedBy=multi-user.target

Reload the daemon and restart the node_exporter

    $ sudo systemctl daemon-reload
    $ sudo systemctl restart node_exporter

Check the status of node_exporter services, and check if the TLS is enabled

    $ sudo systemctl status node_exporter

 Now we have Encryption enabled at the Node exporter level.
 We need to update Prometheus confg file to get metrics from nodes with HTTPS endpoints:
   - Copy the node_exporter.crt file from the node exporter server to the Prometheus server at /etc/prometheus
   
            $ sftp <user on node_exporter>@<ip of node_exporter>
            sftp> get node_exporter.crt
            Fetching /home/rahma/node_exporter.crt to node_exporter.crt
            node_exporter.crt                                                                                      100% 1326   470.3KB/s   00:00
     
            $ sudo cp node_exporter.crt /etc/prometheus
     
   - Update the permission to the CRT file
     
         $ sudo chown prometheus:prometheus node_exporter.crt
     
  Update Prometheus configuration file
  
      $ vi /etc/prometheus/prometheus.yml
         - job_name: 'Linux Server'
           scheme: https
           tls_config:
             ca_file: /etc/prometheus/node_exporter.crt
             insecure_skip_verify: true
           static_configs:
             - targets: ['192.168.56.125:9100']
     
 2) Authentication:
    When setting up authentication with Prometheus, we first have to create a hash of our password.
    I used apache2-utils for generating hash password
    - Install apache2-utils
      
           $ sudo apt-get update && sudo apt install apache2-utils -y

    - generate hash password:
      
          $ htpasswd -nBC 12 "" | tr -d ':\n'
      
    - Go back to your node exporter server, and update the config.yml file at /etc/node_exporter
      
            tls_server_config:
              cert_file: /etc/node_exporter/node_exporter.crt
              key_file: /etc/node_exporter/node_exporter.key
            basic_auth_users:
              prometheus: $2y$12$ES896ZFqiEWkPaT1kFhuE.vTT3NBXCyw9PUgkc27ZtkalgS7jtEiO

      - Restart the node_exporter
        
             $ sudo systemctl restart node_exporter
        
      - we need to update Prometheus conf with username and password at prometheus.yml file with basic_auth
        
              - job_name: 'Linux Server'
                scheme: https
                basic_auth:
                  username: prometheus
                  password: 1234
                tls_config:
                  ca_file: /etc/prometheus/node_exporter.crt
                  insecure_skip_verify: true
                static_configs:
                - targets: ['192.168.56.125:9100']

    - restart the Prometheus server
      
           $ sudo systemctl restart prometheus

** Now we have our Node Exporter UP





















  

    


   

