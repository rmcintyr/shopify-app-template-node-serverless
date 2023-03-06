FROM public.ecr.aws/lambda/nodejs:18

ARG SHOPIFY_API_KEY
ENV SHOPIFY_API_KEY=$SHOPIFY_API_KEY
ARG FUNCTION_DIR="/var/task"
RUN mkdir -p ${FUNCTION_DIR}

WORKDIR ${FUNCTION_DIR}
COPY . ${FUNCTION_DIR}
RUN npm install -g npm@9.6.0
RUN npm install && npm rebuild
RUN cd frontend && npm install && npm run build
CMD ["index.handler"]
