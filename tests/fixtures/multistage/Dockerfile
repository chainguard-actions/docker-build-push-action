FROM alpine:3.18 AS base
RUN echo "base stage" > /stage.txt
CMD ["cat", "/stage.txt"]

FROM base AS final
RUN echo "final stage" >> /stage.txt
CMD ["cat", "/stage.txt"]
