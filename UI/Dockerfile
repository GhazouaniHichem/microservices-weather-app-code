FROM node:20-alpine3.16
WORKDIR /usr/src/app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 3000
HEALTHCHECK --interval=1m --timeout=30s --retries=3 CMD curl --fail http://localhost:3000 || exit 1
RUN addgroup --system admin
RUN adduser --system ghazouanihm --ingroup admin
USER ghazouanihm:admin
ENTRYPOINT ["node"]
CMD [ "app.js" ]