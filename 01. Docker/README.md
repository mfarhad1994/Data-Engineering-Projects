# Docker Data Pipeline

## Table of Contents
- [Understanding Docker and Containers](#understanding_docker_and_containers)
- [Why Use Docker Containers?](#why_use_docker_containers?)
- [Setting Up a Dockerized Environment](#setting_up_a_dockerized_environment)
- [Running Python Inside Docker](#running_python_inside_docker)
- [Persisting Dependencies with a Dockerfile](#persisting_dependencies_with_a_dockerfile)
- [Adding a Python Script to the Container](#adding_a_python_script_to_the_container)
- [Passing Command-Line Arguments to the Script](#passing_command-line_arguments_to_the_script)
- [Conclusion](#conclusion)


## Understanding Docker and Containers

Docker is a platform for packaging software into standardized units called **containers**. Containers are isolated environments, ensuring that applications run consistently regardless of where they are deployed.

For instance, consider a **data pipeline** that processes CSV files. This pipeline might perform data cleaning, transformation, and processing before producing an output dataset. By running this pipeline within a **Docker container**, we ensure that it remains independent of external dependencies and is not affected by changes in the host environment.

---

## Why Use Docker Containers?

Docker offers several advantages when running data pipelines:

- **Reproducibility:** Ensures the pipeline runs the same way on different machines.
- **Local Experimentation:** Facilitates quick iterations and testing.
- **Integration Tests (CI/CD):** Verifies that the pipeline produces expected outputs when integrated with a database or other services.
- **Cloud Deployments:** Supports execution on platforms like AWS Batch or Kubernetes.
- **Spark Integration:** Simplifies distributed data processing.
- **Serverless Execution:** Can be used with AWS Lambda, Google Cloud Functions, etc.

### Integration Testing in Docker

Integration tests verify that a pipeline functions correctly by comparing expected and actual outputs. Suppose our data pipeline interacts with a database. We can:

1. Run the pipeline inside a Docker container.
2. Query the database to confirm expected records are present.
3. Ensure that unwanted records are absent. This approach guarantees consistency and correctness.

---

## Setting Up a Dockerized Environment

### Project Structure & Initialization
In the following, we create directories in which we start Visual Studio code. 

```bash
cd ~Farhad_Mustafayev/Documents/data-engineering
git init
mkdir basics_setup
cd basics_setup
mkdir docker_s
cd docker_s
code .
```

### Launching a Docker Container
 


```bash
docker run -it ubuntu bash
```

`ubuntu` is the name of the image we want to run and then `bash` is a command that we want to execute in this image. everything that comes after the image name
is parameter to this container. The `-it` flag ensures interactive access. 

We can execute bash prompts Inside the container, list directories:

```bash
ls
```

If we remove everything in container and run it again, we are again back to the initial state. So, container is not affected by anything we did previously, this is what isolation actually means.

To delete everything inside a running container (dangerous command):


```bash
rm -rf / --no-preserve-root
```


To exit Docker:

```bash
exit
```

---

## Running Python Inside Docker

```bash
docker run -it python:3.9
```

Once inside Python:

```python
print('hello world')
```

To install external libraries like **pandas**, we need to exit Python (`Ctrl + D`), launch **bash** instead, and install packages:
We need to get to **bash** to be able to install a command and for that we need to overwrite the **entrypoint**. Entrypoint is what exactly is executed when we run this container and entrypoint can be bash. Now instead of having a python prompt, we have a bash prompt and we can execute bash commands, and now we are installing pandas on this specific docker container:
```bash
docker run -it --entrypoint=bash python:3.9
pip install pandas
```

After installing, verify it:

```bash
python
import pandas
pandas.__version__
```

However, if we exit and re-enter, **pandas will not persist**, as Docker containers reset to their base image.

---

## Persisting Dependencies with a Dockerfile

To ensure `pandas` is available in all future container runs, we create a **Dockerfile**:

```Dockerfile
FROM python:3.9
RUN pip install pandas
ENTRYPOINT ["bash"]
```

Now we can build this docker file. The image name will be **test**. We want docker to build an image in the current directory with **.** . It will look for docker file and it will execute this docker file. Build the image:

```bash
docker build -t test:pandas .
```

Run the newly built container:

```bash
docker run -it test:pandas
```

This container now permanently includes `pandas`.

---

## Adding a Python Script to the Container

Create a script (`pipeline.py`):

```python
import pandas as pd
print('Job finished successfully')
```

Modify the Dockerfile to copy the script into the container. We can also specify the working directory so work directory. This will be the location (/app) in the image in the container where we will copy the file:

```Dockerfile
FROM python:3.9
RUN pip install pandas
WORKDIR /app
COPY pipeline.py pipeline.py
ENTRYPOINT ["bash"]
```

Rebuild the image:

```bash
docker build -t test:pandas .
```

Run the container and check the script:

```bash
docker run -it test:pandas
cd /app
ls
python pipeline.py
```

Output:

```
Job finished successfully
```

---

## Passing Command-Line Arguments to the Script

Modify `pipeline.py` to accept an argument:

```python
import sys
import pandas as pd
print(sys.argv)
day = sys.argv[1]
print(f'Job finished successfully for day = {day}')
```

Update the Dockerfile to execute `pipeline.py` directly:

```Dockerfile
FROM python:3.9
RUN pip install pandas
WORKDIR /app
COPY pipeline.py pipeline.py
ENTRYPOINT ["python", "pipeline.py"]
```

Rebuild and run the pipeline with an argument 2025-03-02:

```bash
docker build -t test:pandas .
docker run -it test:pandas 2025-03-02
```

Expected output:

```
['pipeline.py', '2025-03-02']
Job finished successfully for day = 2025-03-02
```

---

## Conclusion

By leveraging Docker, we ensure:

- **Consistent execution** of our data pipeline.
- **Easy dependency management**.
- **A reproducible environment** for testing and cloud deployments.

This approach is fundamental for scalable and maintainable data engineering workflows.
