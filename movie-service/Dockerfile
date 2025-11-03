FROM python:3.8-slim

# Pas de .pyc et logs non-bufferisés
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

WORKDIR /app

# Installer les dépendances système minimales
RUN apt-get update \
    && apt-get install -y gcc \
    && apt-get clean

# Copier et installer les dépendances Python
COPY requirements.txt .
RUN pip install --no-cache-dir --upgrade pip \
 && pip install --no-cache-dir -r requirements.txt

# Copier le code de l’application
COPY app ./app

# Exposer le port utilisé par FastAPI
EXPOSE 8000

# Démarrer le serveur Uvicorn
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--loop", "asyncio"]
