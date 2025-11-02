# Image Python légère et stable
FROM python:3.9-slim

# Éviter les .pyc et forcer le flush des logs
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

# Dossier de travail
WORKDIR /app

# Dépendances système minimales (compilation éventuelle)
RUN apt-get update \
    && apt-get install -y gcc \
    && apt-get clean

# Installer les dépendances Python
COPY requirements.txt .
RUN pip install --no-cache-dir --upgrade pip \
 && pip install --no-cache-dir -r requirements.txt

# Copier le code de l’application
COPY app ./app

# Port interne exposé par l’API
EXPOSE 8000

# Lancer l’API FastAPI avec Uvicorn (même que dans docker-compose)
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
