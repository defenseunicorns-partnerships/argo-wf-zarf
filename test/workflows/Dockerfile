FROM ubuntu:latest

WORKDIR /app/path
USER root

RUN apt-get update -y && apt-get upgrade -y \
   && apt-get install vim -y \
   && apt-get install curl -y

RUN curl https://dl.min.io/client/mc/release/linux-amd64/mc \
   -o /usr/bin/mc

RUN mkdir /data
RUN mkdir /data/outputs

RUN chown -R 1001:1001 /app/path
RUN chown -R 1001:1001 /data
RUN chmod 755 /app/path
RUN chmod 755 /data

RUN mkdir /.mc
RUN chmod +x /usr/bin/mc
RUN chmod 777 /usr/bin/
RUN chmod 777 /.mc

USER 1001
CMD ["sleep", "infinity"]
