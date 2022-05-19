ARG GO_IMG_TAG=alpine
FROM golang:${GO_IMG_TAG}

COPY --chown=nobody:nobody ./app.go /app/
ENV GOCACHE=/app/.cache/go-build
USER nobody:nobody
WORKDIR /app/

CMD [ "go", "run", "app.go" ]