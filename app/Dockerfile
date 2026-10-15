FROM python:3.12-slim

RUN groupadd --gid 10001 app \
    && useradd \
       --uid 10001 \
       --gid 10001 \
       --no-create-home \
       --shell /usr/sbin/nologin \
       app



WORKDIR /app

COPY --chown=10001:10001 app.py /app/app.py

USER 10001:10001

EXPOSE 8080

HEALTHCHECK --interval=30s \
            --timeout=3s \
            --start-period=10s \
            --retries=3 \
    CMD ["python", "-c", "import urllib.request; urllib.request.urlopen('http://127.0.0.1:8080/healthcheck', timeout=2)"]

CMD ["python", "-u", "/app/app.py"]

