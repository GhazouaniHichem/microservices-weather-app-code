FROM python:3.12.0a7-alpine3.18
WORKDIR /app
COPY requirements.txt requirements.txt
RUN pip3 install -r requirements.txt
COPY . .
EXPOSE 5000
HEALTHCHECK --interval=1m --timeout=30s --retries=3 CMD curl --fail http://localhost:5000 || exit 1
RUN addgroup --system admin
RUN adduser --system ghazouanihm --ingroup admin
USER ghazouanihm:admin
ENTRYPOINT ["python3"]
CMD [ "/app/main.py" ]