# The deployable artifact: code + dependencies + data, frozen in one image.
FROM python:3.12-slim

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY cafe_atlas/ cafe_atlas/
COPY data/raw/ data/raw/

# GIT_SHA is passed at run time (docker run -e GIT_SHA=...) and printed in the report
ENTRYPOINT ["python", "-m", "cafe_atlas.pipeline"]
