# Bharat-Hiring
1. Setting Up the Project
Initialize a Django project (django-admin startproject faq_project).
Create an app (python manage.py startapp faq).
Install necessary dependencies:
pip install django djangorestframework django-ckeditor googletrans==4.0.0-rc1 redis django-redis flake8 pytest
Designing the FAQ Model
Define the FAQ model in models.py:
from django.db import models
from ckeditor.fields import RichTextField

class FAQ(models.Model):
    question = models.TextField()
    answer = RichTextField()
    language = models.CharField(max_length=5, choices=[('en', 'English'), ('hi', 'Hindi'), ('bn', 'Bengali')])

    def get_translation(self, lang):
        return getattr(self, f'question_{lang}', self.question)

    def __str__(self):
        return self.question
3. WYSIWYG Editor Integration
Add ckeditor to INSTALLED_APPS.
Configure it in settings.py
INSTALLED_APPS = [
    "ckeditor",
    "faq",
    "rest_framework",
]

CKEDITOR_CONFIGS = {
    "default": {
        "toolbar": "full",
        "height": 300,
        "width": "100%",
    }
}
4. API Development with Multilingual Support
Create a serializer:
from rest_framework import serializers
from .models import FAQ

class FAQSerializer(serializers.ModelSerializer):
    class Meta:
        model = FAQ
        fields = ["id", "question", "answer", "language"]
Implement the API view:
from rest_framework.response import Response
from rest_framework.views import APIView
from django.core.cache import cache
from .models import FAQ
from .serializers import FAQSerializer

class FAQView(APIView):
    def get(self, request):
        lang = request.GET.get("lang", "en")
        cache_key = f"faqs_{lang}"
        data = cache.get(cache_key)

        if not data:
            faqs = FAQ.objects.filter(language=lang)
            serializer = FAQSerializer(faqs, many=True)
            data = serializer.data
            cache.set(cache_key, data, timeout=3600)  # Cache for 1 hour

        return Response(data)
 Implementing Caching with Redis
Install Redis:
sudo apt install redis
Configure settings.py:
CACHES = {
    "default": {
        "BACKEND": "django_redis.cache.RedisCache",
        "LOCATION": "redis://127.0.0.1:6379/1",
        "OPTIONS": {
            "CLIENT_CLASS": "django_redis.client.DefaultClient",
        }
    }
}
 Multi-Language Translation Support
Use googletrans to translate during creation:
from googletrans import Translator

def translate_faq(instance):
    translator = Translator()
    instance.question_hi = translator.translate(instance.question, dest="hi").text
    instance.question_bn = translator.translate(instance.question, dest="bn").text
    instance.save()
7. Admin Panel Integration
Register the model:
from django.contrib import admin
from .models import FAQ

@admin.register(FAQ)
class FAQAdmin(admin.ModelAdmin):
    list_display = ("question", "language")
Unit Testing with pytest
Install pytest-django:
pip install pytest-django
Create tests.py:
import pytest
from django.urls import reverse
from .models import FAQ

@pytest.mark.django_db
def test_faq_list(api_client):
    response = api_client.get(reverse("faqs"))
    assert response.status_code == 200
