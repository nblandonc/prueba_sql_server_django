# prueba_sql_server_django

# Guía Paso a Paso: Integración de Django con SQL Server para Control de Permisos

## 1. Instalación y Configuración

### 1.1 Instalación de Dependencias
Ejecutar en la terminal:
```bash
pip install django djangorestframework pyodbc djangorestframework-simplejwt
```

### 1.2 Configurar SQL Server en `settings.py`
```python
DATABASES = {
    'default': {
        'ENGINE': 'mssql',
        'NAME': 'buy_n_large_db',
        'USER': 'sa',
        'PASSWORD': 'TuContraseña',
        'HOST': 'localhost',
        'PORT': '1433',
        'OPTIONS': {'driver': 'ODBC Driver 17 for SQL Server'},
    }
}
```

## 2. Definir Modelos en `models.py`
```python
from django.db import models

class Usuario(models.Model):
    nombre = models.CharField(max_length=100)
    usuario = models.CharField(max_length=50, unique=True)
    contraseña_hash = models.BinaryField()

class Rol(models.Model):
    nombre = models.CharField(max_length=50, unique=True)

class UsuarioRol(models.Model):
    usuario = models.ForeignKey(Usuario, on_delete=models.CASCADE)
    rol = models.ForeignKey(Rol, on_delete=models.CASCADE)

class PermisoTabla(models.Model):
    rol = models.ForeignKey(Rol, on_delete=models.CASCADE)
    tabla_nombre = models.CharField(max_length=50)
    permiso_tipo = models.CharField(max_length=10, choices=[('LECTURA', 'LECTURA'), ('ESCRITURA', 'ESCRITURA')])

class PermisoRegistro(models.Model):
    usuario = models.ForeignKey(Usuario, null=True, blank=True, on_delete=models.CASCADE)
    rol = models.ForeignKey(Rol, null=True, blank=True, on_delete=models.CASCADE)
    tabla_nombre = models.CharField(max_length=50)
    registro_id = models.IntegerField()
    permiso_tipo = models.CharField(max_length=10, choices=[('LECTURA', 'LECTURA'), ('ESCRITURA', 'ESCRITURA')])
```

## 3. Crear Serializadores en `serializers.py`
```python
from rest_framework import serializers
from .models import Usuario, Rol, PermisoTabla, PermisoRegistro

class UsuarioSerializer(serializers.ModelSerializer):
    class Meta:
        model = Usuario
        fields = '__all__'

class RolSerializer(serializers.ModelSerializer):
    class Meta:
        model = Rol
        fields = '__all__'

class PermisoTablaSerializer(serializers.ModelSerializer):
    class Meta:
        model = PermisoTabla
        fields = '__all__'

class PermisoRegistroSerializer(serializers.ModelSerializer):
    class Meta:
        model = PermisoRegistro
        fields = '__all__'
```

## 4. Crear Vistas con DRF en `views.py`
```python
from django.db import connection
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status

class ObtenerDatosView(APIView):
    def get(self, request, tabla_nombre):
        usuario_id = request.user.id
        with connection.cursor() as cursor:
            cursor.callproc("ObtenerDatosConPermisos", [usuario_id, tabla_nombre])
            columnas = [col[0] for col in cursor.description]
            datos = [dict(zip(columnas, row)) for row in cursor.fetchall()]
        return Response(datos, status=status.HTTP_200_OK)

class ModificarRegistroView(APIView):
    def post(self, request, tabla_nombre, registro_id):
        usuario_id = request.user.id
        nueva_data = request.data.get("data")
        with connection.cursor() as cursor:
            cursor.callproc("ModificarRegistro", [usuario_id, tabla_nombre, registro_id, nueva_data])
        return Response({"message": "Registro modificado (si tienes permisos)"}, status=status.HTTP_200_OK)
```

## 5. Definir Rutas en `urls.py`
```python
from django.urls import path
from .views import ObtenerDatosView, ModificarRegistroView

urlpatterns = [
    path('datos/<str:tabla_nombre>/', ObtenerDatosView.as_view(), name='obtener_datos'),
    path('modificar/<str:tabla_nombre>/<int:registro_id>/', ModificarRegistroView.as_view(), name='modificar_registro'),
]
```

## 6. Configurar Autenticación con JWT en `settings.py`
```python
INSTALLED_APPS += ['rest_framework', 'rest_framework_simplejwt']

REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': (
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    )
}
```

## 7. Generar Tokens de Usuario
```python
from rest_framework_simplejwt.tokens import RefreshToken
from django.contrib.auth import get_user_model

User = get_user_model()

def generar_token(usuario):
    usuario_obj = User.objects.get(usuario=usuario)
    refresh = RefreshToken.for_user(usuario_obj)
    return {'refresh': str(refresh), 'access': str(refresh.access_token)}
```

## 8. Pruebas con cURL
### Obtener datos con permisos
```bash
curl -X GET http://127.0.0.1:8000/api/datos/empleados/ \
     -H "Authorization: Bearer <TOKEN>"
```

### Modificar un registro (si tiene permiso)
```bash
curl -X POST http://127.0.0.1:8000/api/modificar/empleados/1/ \
     -H "Authorization: Bearer <TOKEN>" \
     -H "Content-Type: application/json" \
     -d '{"data": "Nuevo valor"}'
```

## 9. Conclusión
✔ **SQL Server maneja la lógica de permisos**  
✔ **Django Rest Framework solo expone los datos con restricciones aplicadas**  
✔ **Autenticación con JWT para seguridad**

