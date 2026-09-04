FROM apache/spark:4.2.0-scala2.13-java17-python3-ubuntu AS spark-python

FROM apache/airflow:3.3.1-python3.12

# Reuse the Python client shipped by the Spark Connect server image.  Besides
# avoiding a second 450 MB source download, this keeps the client and server on
# exactly the same Spark/py4j versions.
COPY --from=spark-python --chown=airflow:root /opt/spark/python /opt/spark/python
ENV PYTHONPATH=/opt/spark/python:/opt/spark/python/lib/py4j-0.10.9.9-src.zip

COPY --chown=airflow:root docker/airflow/requirements.txt /tmp/lab28-requirements.txt
RUN pip install --no-cache-dir -r /tmp/lab28-requirements.txt

COPY --chown=airflow:root pyproject.toml README.md /opt/lab28/
COPY --chown=airflow:root src /opt/lab28/src
RUN pip install --no-cache-dir --no-deps /opt/lab28
