FROM library/postgres

RUN apt-get update && \
    apt-get upgrade && \
    apt-get install -y postgis

ENV POSTGRES_USER fcc2
ENV POSTGRES_PASSWORD fcc2
ENV POSTGRES_DB fcc2

COPY postgresql.conf /etc/postgresql/postgresql.conf
