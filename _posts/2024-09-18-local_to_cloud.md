---
title: "From Local to Cloud - Mastering ML Model Deployment"
excerpt: ""
categories:
  - Blog
---

# 1. Introduction

Deploying machine learning (ML) models in production is a crucial step toward delivering real value from data-driven solutions. As machine learning engineers or data scientists, the ability to make these models available to users through an API or service is essential.

Major cloud providers offer a range of services to help you deploy your ML models at different levels of abstraction. These services provide managed infrastructure, scalability, and monitoring, allowing you to focus on the model itself rather than the surrounding infrastructure. But these come with their own set of challenges and trade-offs.

Each solution makes it easier to deploy your models than implementing it from scratch, but it comes at the cost of offuscating the actual logic underneath. This can make debugging and monitoring harder, as you have less visibility into the implementation details. Therefore, I believe it is a great exercise to go through the different levels of abstraction when deploying ML models, as it will give you a better understanding of the trade-offs and challenges at each level.

In this article, we'll explore the journey from deploying an ML model locally using FastAPI, to containerizing it with Docker, and finally, scaling the deployment using Google Cloud Platform (GCP). Specifically, we’ll dive into Google’s Vertex AI (formerlly known as AI Platform) and the different options it provides—from fully managing custom containers to using prebuilt prediction services.

# 2. The Journey of ML Model Deployment: From Local to Cloud

Serving an ML model can be made at different levels of abstraction. The lowest level involves deploying the model directly on your local machine. This gives you full control but comes with the cost of having to implement the server logic and challenges around scalability and infrastructure management.

As you progress up the abstraction ladder, GCP offers solutions that take away much of the complexity, allowing you to focus on the model itself rather than the surrounding infrastructure.

We'll explore three key stages in this journey:

1. Local Deployment with FastAPI.
2. Containerizing the model Server with Docker.
3. Deploying on Google Cloud AI Platform with various abstraction levels.

# 3. Level 1: Local Deployment with FastAPI

The first step is to deploy your ML model locally using FastAPI, a modern, fast (high-performance) web framework for Python (Although other alternatives could be used such as Flask or Django). This low-level approach gives you full control over the prediction logic but requires manually implementing all the server logic. It's a good starting point for testing and development but would not be appropriate for production deployments as this does not provide scalability or fault tolerance.

> **Note**: In this article, we will not cover the training of the machine learning model. However, for demonstration purposes, we will be using an [XGBoost model](https://xgboost.readthedocs.io/en/stable/) trained on the classic [Iris dataset](https://scikit-learn.org/stable/auto_examples/datasets/plot_iris_dataset.html). The complete training process can be found in the Jupyter Notebook linked in [section 7](#7-jupyter-notebook-full-steps-for-each-level).

Assuming you have already installed FastAPI, you now need to create a python script that instanteates a FastAPI app, loads the trained model and define the health and prediction routes using the FastAPI decorators syntax. Below is a simple example of a FastAPI app that does this:

```python
# app/app.py
import numpy as np
import xgboost as xgb
from fastapi import FastAPI, HTTPException, Request
import os

artifact_local_path = ... # path to the trained model artifact

app = FastAPI()

# load model
model_ = xgb.XGBClassifier()
model_.load_model(artifact_local_path)

HEALTH_ROUTE = "/health"
PREDICTIONS_ROUTE = "/predictions"

def preprocess(content):
    preprocessed_content = np.asarray(content)
    return preprocessed_content

def predict(ingest_data):
    return model_.predict(ingest_data)

def postprocess(predictions):
    predictions = predictions.tolist()
    return {"predictions": predictions}

@app.get(HEALTH_ROUTE, status_code=200)
def health():
    return {"Server is up and running!"}


@app.post(PREDICTIONS_ROUTE)
async def predict(request: Request):

    request_json = await request.json()
    content = request_json["instances"]

    ingest_data = preprocess(content)

    predictions = prediction(ingest_data)

    output = postprocess(predictions)

    return output
```

- The FastAPI app is instanteated using `app = FastAPI()`. This is then used to define the health and prediction routes using the FastAPI decorators syntax on functions `health` and `predict`.
- The `predict` function preprocesses the input data, makes predictions using the loaded model, and postprocesses the predictions before returning the output.
- The `health` function returns a simple message to indicate that the server is up and running. Although this is a simple example, you can extend this to include more complex health checks.

The app can be run using the `fastapi run` command. This will start the FastAPI server on the default port 8000. You can then send a POST request to the `/predictions` route with the input data to get the predictions. Bellow is an example of how you can do this using the `requests` library:

```python
import requests

data = {"instances": ...} # input data

response = requests.post("http://localhost:8000/predictions", json=data)
```

### Pros:

- Full control over the prediction logic.
- Easy debugging and testing in a local environment.

### Cons:

- Lack of scalability and fault tolerance.
- Requires manual setup of the environment, hindering reproducibility.

# 4. Level 2: Containerizing the FastAPI Server with Docker

Once your FastAPI server is running locally, the next step is to package it into a Docker container for portability and ease of deployment. This intermediate level of abstraction makes your application portable across different environments and will allow you to deploy it locally or in the cloud as we will see in the following section.

To containerize the FastAPI app, you need to create a `Dockerfile` that specifies the base image, copies the app code, installs the necessary dependencies, and runs the FastAPI server. Here is the  Dockerfile that packages the FastAPI app into a container:

```
# Dockerfile
FROM python:3.10
COPY ./app /app # copy the FastAPI app to the container
COPY ./model-artifacts /model-artifacts # copy the trained model artifacts to the container

RUN pip install -r /app/requirements.txt # install the dependencies

CMD ["fastapi", "run"]
EXPOSE 8000
```

- The `Dockerfile` specifies the base image as `python:3.10` and copies the FastAPI app and model artifacts to the container. The `requirements.txt` file contains the dependencies needed to run the FastAPI app.
- The `CMD` instruction specifies the command to run when the container starts, which in this case is `fastapi run`. The `EXPOSE` instruction exposes port 8000, which is the default port used by FastAPI.
- You can then build the Docker image using the `docker build` command and run the container using the `docker run` command.

### Pros:

- Portability across any platform that runs Docker.
- Simplifies deployment on other environments, including cloud platforms.
- Cloud agnostic, allowing you to deploy the container on any cloud provider with little modification.

### Cons:

- Still requires you to manage the infrastructure manually, although containers offer better portability and scalability.

# 5. Level 3: Vertex AI Prediction

The final stage involves moving the model to Vertex AI . This platform allows for scalable deployment while taking away much of the complexity of managing the infrastructure. We'll explore three approaches under this section: deploying custom containers, using custom prediction routines, and utilizing prebuilt containers.

> **Note:** We could also deploy the model directly to Google Cloud Run, which is a fully managed platform that automatically scales your containerized applications, or use Google Kubernetes Engine.

Using Vertex AI provides convenient features specific to ML models, such as model monitoring, explainability, and versioning. Moreover it allows for easy testing through straightforward batch prediction jobs or online prediction endpoints

Models deployed on Vertex AI first need to be registered in the Model Registry, this logically groups all the necessary prerequisites to run the model server, such as the image, the model artifacts and the infrastructure. The following sections will cover the different alternatives available on Vertex AI.

## 5.1 Custom Container Deployment

In this scenario, you take the Docker container created in the previous step and use Vertex AI to serve it. This gives you control over the environment, while GCP manages scaling and infrastructure.

### Steps:

1. **Push Docker image to Artifact Registry**: To push the Docker image to Artifact Registry, begin by creating a repository using the Google SDK command `gcloud artifacts repositories create`. Next, configure the Docker client to authenticate with Artifact Registry by running `gcloud auth configure-docker`. Once authenticated, tag your image appropriately using the `docker tag` command, ensuring the tag includes the Artifact Registry location and repository name. Finally, push the tagged image to the repository with the `docker push` command, which will upload your container image to Artifact Registry, making it available for deployment or further use within your Google Cloud environment.

2. **Add to Model Registry**: Model Registry is a service in Vertex AI that allows you to manage your models. Once the model is registered, you can deploy it to an endpoint or start a batch prediction job.

    To register the model you can use the Python SDK as follows:

    ```python
    from google.cloud import aiplatform

    model = aiplatform.Model.upload(
        project=...,
        location=...
        display_name=...,
        serving_container_image_uri=..., # URI of the Docker image in Artifact Registry
        serving_container_predict_route="/predictions",
        serving_container_health_route="/health",
        serving_container_ports=[8000],
    )

    model.wait()
    ```

3. **Test the model**: Once you have registered the model you can test it either by deploying it to an endpoint or by making a batch prediction call to the model. Below is an example of the former:
    
    ```python
    batch_prediction_job = model.batch_predict(
    job_display_name="iris_batch_prediction",
    bigquery_source=bigquery_source_input_uri,
    bigquery_destination_prefix=bigquery_destination_output_uri,
    machine_type="e2-standard-2",
    starting_replica_count=1,
    max_replica_count=1,
    )

    batch_prediction_job.wait()
    ```
    This will spin up a GCE instance and run the prediction job, then once finished it will shut down the instance, and therefore you will be charged only for the necessary resources. Note that batch prediction only accept input data from GCS or BigQuery, in this case we are using BigQuery as the source of the input data.

### Pros:

- Full control over the environment and runtime (e.g., libraries, dependencies).
- Full Control as well over the prediction logic, which may be useful for instance for custom pre/post-processing.
- GCP handles scaling and infrastructure management.

### Cons:

- You still manage the container and are responsible for maintaining it.
- Requires more effort as you need to implement the prediction logic, set up the environment and manage the container.

## 5.2 Custom Prediction Routine

Vertex AI offers Custom Prediction Routines through the Python SDK. This allows you to define only the prediction logic without any need for the server implementation or containerization. This is a higher level of abstraction compared to deploying custom containers which comes with its own set of benefits and drawbacks.

### Steps:

1. **Write a Predictior**: This essentially comes down to defining a "Predictor" class that implements the `preprocess`, `prediction` and `postprocess` functions we previously implemented in the FastAPI app. Below is an example of how you can define a "Predictor" for an XGBoost model:

    ```python
    # predictor.py
    import numpy as np
    from google.cloud.aiplatform.prediction.sklearn.predictor import SklearnPredictor
    from google.cloud.aiplatform.utils import prediction_utils
    import xgboost as xgb
    import os

    class CprPredictor(SklearnPredictor):
        
        def __init__(self):
            return
        
        def load(self, artifacts_uri: str):
            """Loads the preprocessor artifacts."""
            prediction_utils.download_model_artifacts(artifacts_uri)

            self._model = xgb.XGBClassifier()
            self._model.load_model("model.json")
        
        def preprocess(self, prediction_input):
            instances  = np.asarray(prediction_input["instances"])
            return np.asarray(instances)

        def predict(self, inputs):
            prediction_results = self._model.predict(inputs)
            return prediction_results
        
        def postprocess(self, prediction_results):
            predictions = prediction_results.tolist()
            return {"predictions": predictions}
    ```
2. **Instanteate a local model**: Instantiate the Predictor class using the following code:

    ```python
    from google.cloud.aiplatform.prediction import LocalModel
    from predictor import CprPredictor # This will depend on the path to the predictor file

    local_model = LocalModel.build_cpr_model(
        cpr_folder,
        image_uri,
        base_image="python:3.10",
        predictor=CprPredictor,
        requirements_path=..., # path to the requirements file
    )
    ```
    This will create a server using the predictor and containerize it using docker. The `requirements_path` is the path to the requirements file that contains the dependencies needed to run the predictor.

3. **Test the model**: To test the model on Vertex AI, you will need to push the image to Artifact Registry and then add it to Model Registry. Then you can deploy it, or in this case, run a batch prediction job. Below is an example of how you can do this:
        
    ```python
    local_model.push_image()

    model = aiplatform.Model.upload(
    project=...,
    location=...,
    local_model=local_model,
    display_name=...,
    artifact_uri=bucket_cpr_path # GCS path to the model artifacts (local path is not supported here)
    )

    ... # Run a batch prediction job as previously described
    ```

### Pros:

- Allows complete customization of the prediction logic.
- No need to worry about defining an HTTP server or containerization.

### Cons:

- Requires more effort than deploying with prebuilt containers.
- Offuscated implementation details which may make debugging harder.

## 5.3 Using Vertex AI Precition Prebuilt Containers

The highest level of abstraction is to use Vertex AI prediction prebuilt containers for model serving. Here, you don’t need to worry about managing the container or defining custom routines. You only need to upload the trained model artifact and register to Vertex AI Model Registry.

Each model type (e.g., TensorFlow, scikit-learn) has its own prebuilt container that you can use to deploy your model. And each container has a specific set of [requirements and constraints](https://cloud.google.com/vertex-ai/docs/training/exporting-model-artifacts) that you need to follow when creating the model artifacts. Mainly in terms of the model format and the artifacts that need to be included in the model directory as well as the compatibility of the model library and the version in the prebuilt container.

### Steps:

1. **Upload model to Cloud Storage**: Store your trained model in a GCP bucket. Here you need to make sure to use the same format and version as expected by the prebuilt container.

    For instance, if you are using a XGBoost model, you need to make sure that the model is saved in the booster format and that the same version of `xgboost` is used. In this case we will be using the `xgboost-cpu.1-7` prebuilt container.
2. **Deploy with prebuilt container**: Use Vertex AI's prebuilt container for the model type (XGBoost in this case) and create a model version.

    ```python	
    # use a gcp vertex prebuilt container 
    from google.cloud import aiplatform

    model = aiplatform.Model.upload(
        project=...,
        location=...,
        display_name=...,
        serving_container_image_uri="us-docker.pkg.dev/vertex-ai/prediction/xgboost-cpu.1-7:latest",
        parent_model=model_resource_name, # This allows us to upload this model as a new version of the existing model
        artifact_uri=booster_model_uri, # GCS path to the model artifacts
    )

    ... # Run a batch prediction job as previously described
    ```

### Pros:

- No need to write any prediction and server logic or to containerize the model.
- Quickest way to deploy a model.

### Cons:

- Less flexibility compared to custom containers or routines.
- Limited to the model types supported by the prebuilt containers.
- Debugging may be harder as the implementation details are not accessible in an straightforward manner.

# 6. Comparison

Here’s a quick comparison of the alternatives discussed in this article based on different criteria:

| Feature                  | FastAPI Local | Docker Container Local | Vertex AI Custom Container | Vertex AI CPR | Vertex AI Prebuilt Container |
|--------------------------|---------------|------------------------|----------------------------|---------------|------------------------------|
| Ease of Use              | Low           | Medium                 | Medium                     | Medium        | High                         |
| Scalability              | Low           | Medium                 | High                       | High          | High                         |
| Infrastructure Management| High          | Medium                 | Low                        | Low           | Low                          |
| Flexibility              | High          | High                   | Medium                     | Medium        | Low                          |
| Cost                     | Low           | Low                    | Medium                     | Medium        | Medium                       |

# 7. Jupyter Notebook: Full steps for each level

To help you get started with deploying your ML model, [here](https://github.com/grudloff/grudloff.github.io/blob/master/assets/notebooks/local_to_cloud.ipynb) is a Jupyter Notebook that walks you through all the necessary steps. This notebook covers:

1. Setup
2. Training the Model
3. Local server
4. Containerization with Docker
5. Using the custom container in the cloud
6. Custom Prediction Routine
7. Prebuilt Container
8. Validate prediction concistency
9. Clean Up

This notebook provides code snippets, explanations, and commands to help you understand and execute each level described in this article in full detail.

# 8. Conclusion

Implementing ML model deployment from the ground up offers several key advantages:

1. Deep Understanding: Building from scratch provides insight into each component's role and interaction.
2. Troubleshooting Skills: Familiarity with lower-level implementations enhances debugging abilities across all abstraction levels.
3. Customization: Lower-level knowledge enables fine-tuning of higher-level solutions when needed.
4. Informed Decision-Making: Experience with different abstraction levels helps in choosing the right deployment strategy for each project.
5. Vendor Flexibility: Starting with local implementations and Docker provides a foundation adaptable to various cloud providers or on-premises solutions.

While cloud platforms offer convenient abstractions, the knowledge gained from lower-level implementations remains invaluable throughout your ML deployment journey.

While this article focuses on Google Cloud Platform (GCP), it's important to note that other cloud providers such as AWS and Azure offer similar services for deploying machine learning models. The exercise of moving from low to high levels of abstraction is equivalent across these providers. For instance, AWS offers services like SageMaker for managed model deployment, and Azure provides Azure Machine Learning for similar capabilities. The principles and steps outlined here can be adapted to fit the offerings of any major cloud provider.

**Key takeaways:**
- Custom Containers offer full control over the environment and runtime, but require more effort to set up and maintain.
- Even though CPR provide full customization over the prediction logic, debugging may be harder than directly implementing a model server as the implementation details are offuscated.
- Using a prebuilt container requires you to take extra care in ensuring that the model artifacts are compatible with the prebuilt container. However, the prediction output from the prebuilt container has the added benefit of being compatible with other GCP services such as explainability and monitoring.
- Making the effort of matching your implementation output with the prebuilt container output can be a good idea in the long run. Having said that, there are scenarios in which this may not be possible, for instance due to no prebuilt image matching your library requirements. One example would be when implementing a model that requires a scikit-learn wrapper around an xgboost model.

I hope this has been enlightening and that you now have a better understanding of the different levels of abstraction when deploying ML models. If you have any questions or feedback, feel free to reach out!

# 9. Bonus: Tips and tricks

- Activate logging when there are issues, this will help you debug in case something goes wrong. You can do this by adding the following snippet at the beginning of your script.
    ```python	
    import logging
    logging.basicConfig(level=logging.INFO)
    ```
    >**Note**: In case you want to do this in the FastAPI app, you would need to add this to the `app.py` file, as activating logging locally in your notebook would not have any effect.
- When building an image multiple time due to debugging. Use the `--no-cache` flag when building the Docker image to ensure that the latest changes are included in the image. For the CPR you can set this trough the `no_cache` parameter in the `LocalModel.build_cpr_model` method.
- In case your model is failing in the cloud, you can deploy it locally using the `LocalModel.deploy_to_local_endpoint` method. This will allow you to test the model locally and debug it before deploying it to the cloud. You can check the examples in the Jupyter Notebook for more details on how to do this for each of the alternatives.
- You can inspect the files in a prebuilt container by running it locally with the above method. Once it is running you can access the container by running `docker exec -it <container_id> /bin/bash`. This will open a shell in the container and you can inspect the files and the environment.