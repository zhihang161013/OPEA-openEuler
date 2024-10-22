# opea-openeuler
OPEA (Open Platform for Enterprise AI) is a framework that enables the creation and evaluation of open, multi-provider, robust, and composable generative AI (GenAI) solutions. 
It harnesses the best innovations across the ecosystem while keeping enterprise-level needs front and center.

OpenEuler is an open source OS oriented to digital infrastructure that fits into any server, cloud computing, edge computing, and embedded deployment.
The openEuler community works with global developers to build an open, diversified, and architecture-inclusive software ecosystem. 
It covers all scenarios of digital facilities, and empowers enterprises to develop their hardware and software as well as application ecosystems.

## Getting started with openEuler on AWS
1. Navigate to [AWS](https://aws.amazon.com/). Click the **services** button at the top right of the screen. Select **Compute** from the options available. Then, head to the **EC2** console to begin the process.
![ec2 console](./pics/ec2-console.png)

2. Within the EC2 console, click the **AMI Catalog** button. Select **Community AMIs** from the options available. In the search bar, type "openEuler" to find the openEuler AMIs available in the community repository.
![ami catalog](./pics/ami-catalog.png)

3. Select an openEuler AMI and click the **Launch Instance with AMI** button.
![community amis](./pics/community-amis.png)

4. Within the instance type list, select the Amazon EC2 M7i or M7i-flex instance type. Complete the remaining configuration as needed. Then, click the **Launch Instance** button.
![configure instance](./pics/configure-instance.png)

Now we have an openEuler cloud instance that fufills the hardware requirements for deploying ChatQnA services.

## Deploying the ChatQnA services
### 1. Setup Environment Variable
To set up environment variables for deploying ChatQnA services, follow these steps:

1. Set the required environment variables:
```shell
# Example: host_ip="192.168.1.1"
export host_ip="External_Public_IP"
# Example: no_proxy="localhost, 127.0.0.1, 192.168.1.1"
export no_proxy="Your_No_Proxy"
export HUGGINGFACEHUB_API_TOKEN="Your_Huggingface_API_Token"
```

2. If you are in a proxy environment, also set the proxy-related environment variables:
```shell
export http_proxy="Your_HTTP_Proxy"
export https_proxy="Your_HTTPs_Proxy"
```
3. Set up other environment variables:
```shell
# on Xeon
source ./set_env.sh
```
### 2. Run Docker Compose
```shell
docker compose -f compose_vllm.openeuler.yaml up -d
```

It will automatically download the docker image on docker hub:
```shell
docker pull openeuler/chatqna:latest
docker pull openeuler/chatqna-ui:latest
...
```
### 3. Validate Microservices
Note, when verify the microservices by curl or API from remote client, please make sure the ports of the microservices are opened in the firewall of the cloud node.
Follow the instructions to validate MicroServices.
For details on how to verify the correctness of the response, refer to how-to-validate_service.
1. TEI Embedding Service

   ```shell
   curl ${host_ip}:6006/embed \
       -X POST \
       -d '{"inputs":"What is Deep Learning?"}' \
       -H 'Content-Type: application/json'
   ```
2. Retriever Microservice

   To consume the retriever microservice, you need to generate a mock embedding vector by Python script. 
   The length of embedding vector is determined by the embedding model. Here we use the model EMBEDDING_MODEL_ID="BAAI/bge-base-en-v1.5", which vector size is 768.

   Check the vector dimension of your embedding model, set your_embedding dimension equals to it.
   ```shell
   export your_embedding=$(python3 -c "import random;   embedding = [random.uniform(-1, 1) for _ in range(768)];    print(embedding)")
   curl http://${host_ip}:7000/v1/retrieval \
     -X POST \
     -d "{\"text\":\"test\",\"embedding\":${your_embedding}}" \
     -H 'Content-Type: application/json'
   ```
3. TEI Reranking Service

   ```shell
   curl http://${host_ip}:8808/rerank \
       -X POST \
       -d '{"query":"What is Deep Learning?", "texts": ["Deep Learning is not...", "Deep learning is..."]}' \
       -H 'Content-Type: application/json'
   ```
4. LLM backend Service

   In first startup, this service will take more time to download the model files.
   After it's finished, the service will be ready.

   Try the command below to check whether the LLM serving is ready.
   ```shell
   docker logs ${CONTAINER_ID} | grep Connected
   ```

   If the service is ready, you will get the response like below.
   ```shell
   2024-09-03T02:47:53.402023Z  INFO text_generation_router::server: router/src/server.rs:2311: Connected
   ```

   Then try the `cURL` command below to validate services.
   ```shell
   # vLLM Service
   curl http://${host_ip}:9009/v1/completions \
       -H "Content-Type: application/json" \
       -d '{"model": "Intel/neural-chat-7b-v3-3", "prompt": "What is Deep Learning?", "max_tokens": 32, "temperature": 0}'
   ```
5. MegaService
   ```shell
   curl http://${host_ip}:8888/v1/chatqna -H "Content-Type: application/json" -d '{
       "messages": "What is the revenue of Nike in 2023?"
       }'
   ```
### 4. Consume the ChatQnA Service
Once the services are up, open the following URL in your browser: http://{host_ip}:5173. 
By default, the UI runs on port 80 internally. 
If you prefer to use a different host port to access the frontend, you can modify the port mapping in the compose_vllm.openeuler.yaml file as shown below:
```shell
chaqna-ui-server:
  image: openeuler/chatqna-ui:latest
  ...
  ports:
    - "80:80"
```

Here is an example of running ChatQnA:
![chatqna-ui](./pics/chatqna-ui.png)
