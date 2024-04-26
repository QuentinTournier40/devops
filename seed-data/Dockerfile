FROM python:3.9-slim

WORKDIR /app

RUN apt-get update && \
    apt-get install -y apache2-utils

COPY . .

RUN python make-data.py

EXPOSE 8000

CMD [ "bash", "generate-votes.sh"]