## settings.py

INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'rest_framework',
    'scraper',
]

CELERY_BROKER_URL = 'redis://localhost:6379/0'  
CELERY_RESULT_BACKEND = 'redis://localhost:6379/0'


## urls.py (url_scraper)
from django.contrib import admin
from django.urls import path,include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/scraper/', include('scraper.urls')),

]

## urls.py(scraper)
from django.urls import path
from . import views

urlpatterns = [
    path('upload/', views.upload_csv, name='upload_csv'),
    path('results/', views.get_scraped_results, name='scraped_results'),

]
## task.py(scraper)

from celery import shared_task
import requests
from .models import MetaData
from requests.exceptions import RequestException


@shared_task
def scrape_url(url):
    try:
        response = requests.get(url, timeout=10)
        
        title = response.text.split('<title>')[1].split('</title>')[0] if '<title>' in response.text else 'No title'
        description = response.text.split('<meta name="description" content="')[1].split('"')[0] if '<meta name="description"' in response.text else 'No description'
        keywords = response.text.split('<meta name="keywords" content="')[1].split('"')[0] if '<meta name="keywords"' in response.text else 'No keywords'

        metadata = MetaData.objects.create(url=url, title=title, description=description, keywords=keywords)
        return {"status": "success", "url": url}
    except RequestException as e:
        return {"status": "error", "url": url, "message": str(e)}

## init.py(url_scraper)
from .celery import app as celery_app

__all__ = ('celery_app')

## celery.py (url_scraper)
from __future__ import absolute_import, unicode_literals
import os
from celery import Celery

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'url_scraper.settings')

app = Celery('url_scraper')
app.config_from_object('django.conf:settings', namespace='CELERY')
app.autodiscover_tasks()

# models.py (scraper)
from django.db import models

class MetaData(models.Model):
    url = models.URLField(unique=True)
    title = models.CharField(max_length=255)
    description = models.TextField()
    keywords = models.TextField()

    def __str__(self):
        return self.url
        
# serializers (scraper)

from rest_framework import serializers
from .models import MetaData

class MetaDataSerializer(serializers.ModelSerializer):
    class Meta:
        model = MetaData
        fields = '__all__'

# views .py (scraper)

from rest_framework.decorators import api_view
from rest_framework.response import Response
from .task import scrape_url
from .models import MetaData
from .serializers import MetaDataSerializer
import csv
from io import StringIO

@api_view(['POST'])
def upload_csv(request):
    if 'file' not in request.FILES:
        return Response({"error": "No file uploaded"}, status=400)

    file = request.FILES['file']
    contents = file.read().decode('utf-8')
    csv_file = StringIO(contents)
    reader = csv.reader(csv_file)
    urls = [row[0] for row in reader]

    # Call Celery task for each URL
    for url in urls:
        scrape_url.delay(url)

    return Response({"message": "URLs are being scraped."}, status=200)

@api_view(['GET'])
def get_scraped_results(request):
    metadata = MetaData.objects.all()
    serializer = MetaDataSerializer(metadata, many=True)
    return Response(serializer.data, status=200)
    
# Docer File
FROM python:3.10-slim-buster

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

ENV PYTHONUNBUFFERED 1

EXPOSE 8000

CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]

# docer-compose.yml

version: "3.8"

services:
  web:
    build: .
    ports:
      - "8000:8000"
    environment:
      - REDIS_HOST=redis
    depends_on:
      - redis
      - db

  redis:
    image: "redis:alpine"

  db:
    image: postgres
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
      POSTGRES_DB: url_scraper

# mains.py

from django.db import models

class MetaData(models.Model):
    url = models.URLField(unique=True)
    title = models.CharField(max_length=255)
    description = models.TextField()
    keywords = models.TextField()

    def __str__(self):
        return self.url





