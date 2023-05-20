FROM python:3.12.0a7-bullseye
WORKDIR /app
COPY requirements.txt requirements.txt
RUN pip3 install -r requirements.txt
COPY . .
EXPOSE 5000
ENTRYPOINT ["python3"]
CMD [ "/app/main.py" ]