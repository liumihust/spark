#
# Licensed to the Apache Software Foundation (ASF) under one or more
# contributor license agreements.  See the NOTICE file distributed with
# this work for additional information regarding copyright ownership.
# The ASF licenses this file to You under the Apache License, Version 2.0
# (the "License"); you may not use this file except in compliance with
# the License.  You may obtain a copy of the License at
#
#    http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#

FROM maven:3.5.4-jdk-8 AS build-env
WORKDIR /spark
ADD . /spark
ENV MAVEN_OPTS="-Xmx2g -XX:ReservedCodeCacheSize=1g"
RUN dev/make-distribution.sh --name dla-spark -Phadoop-2.7 -Phive -Phive-thriftserver -Pkubernetes -T 4 -DskipTests -DrecompileMode=all -X


FROM openjdk:8-jdk-slim

ARG spark_jars=jars
ARG img_path=kubernetes/dockerfiles
ARG k8s_tests=kubernetes/tests

# Before building the docker image, first build and make a Spark distribution following
# the instructions in http://spark.apache.org/docs/latest/building-spark.html.
# If this docker file is being used in the context of building your images from a Spark
# distribution, the docker build command should be invoked from the top level directory
# of the Spark distribution. E.g.:
# docker build -t spark:latest -f kubernetes/dockerfiles/spark/Dockerfile .

RUN set -ex && \
    apt-get update && \
    ln -s /lib /lib64 && \
    apt install -y bash tini libc6 libpam-modules krb5-user libnss3 procps nscd && \
    mkdir -p /opt/spark && \
    mkdir -p /opt/spark/work-dir && \
    touch /opt/spark/RELEASE && \
    rm /bin/sh && \
    ln -sv /bin/bash /bin/sh && \
    echo "auth required pam_wheel.so use_uid" >> /etc/pam.d/su && \
    chgrp root /etc/passwd && chmod ug+rw /etc/passwd && \
    rm -rf /var/cache/apt/*

COPY --from=build-env /spark/dist/${spark_jars} /opt/spark/jars
COPY --from=build-env /spark/dist/bin /opt/spark/bin
COPY --from=build-env /spark/dist/sbin /opt/spark/sbin
COPY --from=build-env /spark/dist/${img_path}/spark/entrypoint.sh /opt/
COPY --from=build-env /spark/dist/examples /opt/spark/examples
COPY --from=build-env /spark/dist/${k8s_tests} /opt/spark/tests
COPY --from=build-env /spark/dist/data /opt/spark/data

# Setup for the Prometheus JMX exporter.
RUN mkdir -p /etc/metrics/conf
# Add the Prometheus JMX exporter Java agent jar for exposing metrics sent to the JmxSink to Prometheus.
ADD https://repo1.maven.org/maven2/io/prometheus/jmx/jmx_prometheus_javaagent/0.12.0/jmx_prometheus_javaagent-0.12.0.jar /prometheus/
COPY resource-managers/kubernetes/docker/src/main/dockerfiles/spark/conf/metrics.properties /etc/metrics/conf
COPY resource-managers/kubernetes/docker/src/main/dockerfiles/spark/conf/prometheus.yaml /etc/metrics/conf


ENV SPARK_HOME /opt/spark

WORKDIR /opt/spark/work-dir

ENTRYPOINT [ "/opt/entrypoint.sh" ]
