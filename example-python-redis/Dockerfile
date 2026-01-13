# syntax=docker/dockerfile:1

# build from python image
FROM python:3.10-alpine

# godo code dir
WORKDIR /code

# set envvars for flask
ENV FLASK_APP=app.py
ENV FLASK_RUN_HOST=0.0.0.0

# install dependencies
RUN apk add --no-cache gcc musl-dev linux-headers
COPY requirements.txt requirements.txt
RUN pip install -r requirements.txt

# listen to port
EXPOSE 5000

# copy app
COPY . .

# set default command for container
CMD ["flask", "run", "--debug"]
