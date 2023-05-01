### Intro
We will deploy MLflow in Mail.ru Cloud Solutions using S3 as artifact store, DBaaS Postgres as backend entity storage and Tracking Server on separate host.   
This is the most production ready scenario of deployment   
https://mlflow.org/docs/latest/tracking.html#scenario-4-mlflow-with-remote-tracking-server-backend-and-artifact-stores


### 1. Create VM for mlflow
https://mcs.mail.ru/help/ru_RU/create-vm/vm-quick-create   
Tested with OS - Ubuntu 18.04   
Record VM white ip and internal ip

### 2. Install Conda on VM created on step 1
You can access VM with command
```console
ssh -i REPLACE_WITH_YOUR_KEY ubuntu@REPLACE_WITH_YOUR_VM_IP
```

Download and install Conda
```console
curl -O https://repo.anaconda.com/archive/Anaconda3-2020.11-Linux-x86_64.sh
bash Anaconda3-2020.11-Linux-x86_64.sh
exec bash
```

### 3. Install MLflow 
We will install MLflow in separate environment
```console
conda create -n mlflow_env
conda activate mlflow_env
conda install python

pip install mlflow
pip install boto3

sudo apt install gcc
pip install psycopg2-binary
```

### 4. Create Postgres as a backend store
https://mcs.mail.ru/help/ru_RU/dbaas-start/db-postgres   
Save credentials, DB name

### 5. Create S3 bucket and directory
The next step is creating a directory for our Tracking Server to log the Machine Learning models and other artifacts. 
Remember that the Postgres database is only used for storing metadata regarding those models.
We will use S3 as artifact storage.

Create bucket and directory inside it  
https://mcs.mail.ru/help/ru_RU/s3-start/create-bucket

Create account, access key id and secret key   
https://mcs.mail.ru/help/ru_RU/s3-start/s3-account 


### 6. Launch MLflow
Login to VM that was created on Step 1   
You can access VM with command  
```console
ssh -i REPLACE_WITH_YOUR_KEY ubuntu@REPLACE_WITH_YOUR_VM_IP
```

Set env variables
```console
sudo nano /etc/environment

#Copy and paste this, replace with your values
MLFLOW_S3_ENDPOINT_URL=https://hb.bizmrg.com
MLFLOW_TRACKING_URI=http://REPLACE_WITH_INTERNAL_IP_MLFLOW_VM:8000
```
Log out, login to apply changes


Create file with S3 credentials
```console
mkdir .aws

nano ~/.aws/credentials
```
Copy and paste this in ~/.aws/credentials
```
[default]
aws_access_key_id = REPLACE_WITH_YOUR_KEY
aws_secret_access_key = REPLACE_WITH_YOUR_SECRET_KEY
```

This is test run, if you leave terminal MLflow will be unaccessible   
Permanent serving of MLflow on the next step   
```console
conda activate mlflow_env
mlflow server --backend-store-uri postgresql://pg_user:pg_password@REPLACE_WITH_INTERNAL_IP_POSTGRESQL/db_name --default-artifact-root s3://REPLACE_WITH_YOUR_BUCKET/REPLACE_WITH_YOUR_DIRECTORY/ -h 0.0.0.0 -p 8000
```

### 7. Enable MLflow as a systemd service
Create dirs for logs and errors
```console
mkdir ~/mlflow_logs/
mkdir ~/mlflow_errors/
```

Create file with systemd service
```console
sudo nano /etc/systemd/system/mlflow-tracking.service
```

Copy and paste in /etc/systemd/system/mlflow-tracking.service
```console
[Unit]
Description=MLflow Tracking Server
After=network.target
[Service]
Environment=MLFLOW_S3_ENDPOINT_URL=https://hb.bizmrg.com
Restart=on-failure
RestartSec=30
StandardOutput=file:/home/ubuntu/mlflow_logs/stdout.log
StandardError=file:/home/ubuntu/mlflow_errors/stderr.log
User=ubuntu
ExecStart=/bin/bash -c 'PATH=/home/ubuntu/anaconda3/envs/mlflow_env/bin/:$PATH exec mlflow server --backend-store-uri postgresql://PG_USER:PG_PASSWORD@REPLACE_WITH_INTERNAL_IP_POSTGRESQL/DB_NAME --default-artifact-root s3://REPLACE_WITH_YOUR_BUCKET/REPLACE_WITH_YOUR_DIRECTORY/ -h 0.0.0.0 -p 8000' 
[Install]
WantedBy=multi-user.target
```

Enable MLflow service
```console
sudo systemctl daemon-reload
sudo systemctl enable mlflow-tracking
sudo systemctl start mlflow-tracking
sudo systemctl status mlflow-tracking
```

Check that evrything is ok in logs
```console
head -n 95 ~/mlflow_logs/stdout.log 
```

### 8. Create JupyterHub host
https://mcs.mail.ru/help/ru_RU/ml-start/ml-info

Login to VM with JupyterHub
```console
ssh -i REPLACE_WITH_YOUR_KEY ubuntu@REPLACE_WITH_YOUR_VM_IP
```

Open tmux and launch JupyterHub
```console
tmux

jupyter-notebook --ip '*'
```
Copy string that looks like that   
http://name_of_host:8888/?token=5d3d6b7a0551asdffds4190e8sdffsd329bee345esdfmkdfs2c042c0b7a5

deattach from tmux session   
ctrl + b d

### 9. Config JupyterHub host
Set env variables

```console
sudo nano /etc/environment

MLFLOW_TRACKING_URI=http://REPLACE_WITH_INTERNAL_IP_MLFLOW_VM:8000
MLFLOW_S3_ENDPOINT_URL=https://hb.bizmrg.com
```
Log out, login to apply changes

Create file with S3 credentials
```console
mkdir .aws

nano ~/.aws/credentials
```
Copy and paste this in ~/.aws/credentials
```
[default]
aws_access_key_id = REPLACE_WITH_YOUR_KEY
aws_secret_access_key = REPLACE_WITH_YOUR_SECRET_KEY
``````

### 10. Install MLflow on JupyterHub host
We will create separate environment, install MLflow and create kernel for JupyterHub with MLflow  
```console
conda create -n mlflow_env

conda activate mlflow_env

conda install python

pip install mlflow

pip install matplotlib
pip install sklearn
pip install boto3

conda install -c anaconda ipykernel

python -m ipykernel install --user --name ex --display-name "Python (mlflow)"
``````

### 11. Launch JupyterHub and test MLflow
Your JupyterHub available on url like that    
http://name_of_host:8888/?token=5d3d6b7a0551asdffds41sdvlgfd8sdffsd329bee345esdfmkdfs2c042c0b7dffb   

You should get this url on step 8   

Launch a terminal in Jupyter and clone the mlflow repo   
```console
git clone https://github.com/stockblog/webinar_mlflow/ webinar_mlflow
```
Open mlflow_demo.ipynb and launch cells


### 12. Serve model from artifact store
Find URI of model in MLFlow UI   
Connect to MLflow host created on step 1   
```console
#EXAMPLE, REPLACE S3 path with your own path to model
mlflow models serve -m s3://mlflow_webinar/artifacts/5/a7ee769713974c118d0eff20226eb474/artifacts/model -h 0.0.0.0 -p 8001

mlflow models serve -m s3://BUCKET/FOLDER/EXPERIMENT_NUMBER/INTERNAL_MLFLOW_ID/artifacts/model -h 0.0.0.0 -p 8001

```

### 13. Serve model from the model registry
Register model in MLflow Models UI. Copy model name and paste it to example string
```console
#EXAMPLE
mlflow models serve -m "models:/diabet_test/Staging"

mlflow models serve -m "models:/YOUR_MODEL_NAME/STAGE"
```

### 14. Test model 
You may need to replace port from 8001 to default port 5001 if you did not set -p parameter in prev step

```console
#EXAMPLE, replace ip adress with internal ip of your mlflow host. You could also use white ip, set firewall settings 
curl -X POST -H "Content-Type:application/json; format=pandas-split" --data '{"columns":["age", "sex", "bmi", "bp", "s1", "s2", "s3", "s4", "s5", "s6"], "data":[[0.0453409833354632, 0.0506801187398187, 0.0606183944448076, 0.0310533436263482, 0.0287020030602135, 0.0473467013092799, 0.0544457590642881, 0.0712099797536354, 0.133598980013008, 0.135611830689079]]}' http://0.0.0.0:8001/invocations

# If you are accessing model from another server, replace IP of MLflow host created on step 1
curl -X POST -H "Content-Type:application/json; format=pandas-split" --data '{"columns":["age", "sex", "bmi", "bp", "s1", "s2", "s3", "s4", "s5", "s6"], "data":[[0.0453409833354632, 0.0506801187398187, 0.0606183944448076, 0.0310533436263482, 0.0287020030602135, 0.0473467013092799, 0.0544457590642881, 0.0712099797536354, 0.133598980013008, 0.135611830689079]]}' http://REPLACE_WITH_IP_MLFLOW_VM:8001/invocations
```

### 15. Permanent serving of model
Connect to MLflow host created on step 1  
you can find MLFLOW_ENV_OF_MODEL when you launch model on step 11 or 12   

sudo nano /etc/systemd/system/mlflow-model.service
```console
[Unit]
Description=MLFlow Model Serving
After=network.target

[Service]
Restart=on-failure
RestartSec=30
StandardOutput=file:/home/ubuntu/mlflow_logs/stdout.log
StandardError=file:/home/ubuntu/mlflow_errors/stderr.log
Environment=MLFLOW_TRACKING_URI=http://REPLACE_WITH_INTERNAL_IP_MLFLOW_VM:8000
Environment=MLFLOW_CONDA_HOME=/home/ubuntu/anaconda3/
Environment=MLFLOW_S3_ENDPOINT_URL=https://hb.bizmrg.com
ExecStart=/bin/bash -c 'PATH=/home/ubuntu/anaconda3/envs/REPLACE_WITH_MLFLOW_ENV_OF_MODEL/bin/:$PATH exec mlflow models serve -m "models:/YOUR_MODEL_NAME/STAGE" -h 0.0.0.0 -p 8001'

[Install]
WantedBy=multi-user.target
```

Then enable new service
```console
sudo systemctl daemon-reload
sudo systemctl enable mlflow-model
sudo systemctl start mlflow-model
sudo systemctl status mlflow-model
```

### 16. Build docker image with model
Prerequisites: install Docker   
https://docs.docker.com/engine/install/ubuntu/   
https://docs.docker.com/engine/install/linux-postinstall/   

You may need to edit conda.yaml and add dependencies   
You can find this file in artifact storage in folder artifacts/model   
s3://BUCKET/FOLDER/EXPERIMENT_NUMBER/INTERNAL_MLFLOW_ID/artifacts/model   

Example conda.yaml where we added scipy and boto3 libs to standart dependencies
```console
channels:
- defaults
- conda-forge
dependencies:
- python=3.6.6
- pip
- pip:
  - mlflow
  - scikit-learn==0.19.2
  - cloudpickle==0.5.5
  - scipy
  - boto3
name: mlflow-env
```

Build docker image with model
```console
#Example
mlflow models build-docker -m "models:/diabet_test/Staging" -n "mlflow-diabetes-model"

mlflow models build-docker -m "models:/YOUR_MODEL_NAME/STAGE" -n "DOCKER_IMAGE_NAME"
```

Launch container with model
```
docker run -p 5001:8080 "mlflow-diabetes-model"
```

Test model in docker
```
curl -X POST -H "Content-Type:application/json; format=pandas-split" --data '{"columns":["age", "sex", "bmi", "bp", "s1", "s2", "s3", "s4", "s5", "s6"], "data":[[0.0453409833354632, 0.0506801187398187, 0.0606183944448076, 0.0310533436263482, 0.0287020030602135, 0.0473467013092799, 0.0544457590642881, 0.0712099797536354, 0.133598980013008, 0.135611830689079]]}' http://127.0.0.1:5001/invocations
```
