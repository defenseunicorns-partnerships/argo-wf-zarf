FROM ubuntu:latest

WORKDIR /app/path
USER root

RUN apt-get update -y && apt-get upgrade -y \
   && apt-get install vim -y \
   && apt-get install curl -y \
   && mkdir /app/path/minio-binaries

COPY mc /app/path/minio-binaries/

#RUN curl https://dl.min.io/client/mc/release/linux-amd64/mc \
   #-o /app/path/minio-binaries/

RUN mkdir /data
RUN mkdir /data/outputs

RUN chown -R 1001:1001 /app/path
RUN chown -R 1001:1001 /data
RUN chmod 755 /app/path
RUN chmod 755 /data

RUN chmod +x /app/path/minio-binaries/mc

USER 1001
CMD ["sleep", "infinity"]
