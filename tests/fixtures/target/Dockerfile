FROM alpine:3.18 AS base
RUN echo "base" > /stage.txt

FROM base AS development
RUN echo "development" > /stage.txt

FROM base AS production
RUN echo "production" > /stage.txt
