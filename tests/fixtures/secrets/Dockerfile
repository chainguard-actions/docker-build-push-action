# syntax=docker/dockerfile:1
FROM alpine:3.18
RUN --mount=type=secret,id=mysecret cat /run/secrets/mysecret > /secret-present.txt 2>/dev/null || echo "no-secret" > /secret-present.txt
CMD ["cat", "/secret-present.txt"]
