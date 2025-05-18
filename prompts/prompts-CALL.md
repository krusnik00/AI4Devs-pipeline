# Prompts Collection

## General Project Analysis

```
Analiza el codebase del proyecto para que tengas un mayor conocimiento
```

## Add New Prompts

```
Quiero que analices #codebase el proyecto para que tengas un mayor conocimiento, tambien quiero que agreges en el documento de prompts.md los prompts que te paso y que tenga formato md
```

## Create New API Endpoints

```
Basándote en las buenas prácticas del manifesto de ManifestoBuenasPracticas.md, dame las instrucciones detalladas para crear un nuevo endpoint GET /position/:id/candidates

Sé específico en los archivos donde va cada código.
Dame la respuesta en español.

Contexto del proyecto:
backend
```

## AWS EC2 Setup Instructions

```
Explica detalladamente cómo configurar una instancia EC2 para desplegar esta aplicación, incluyendo configuración de seguridad, instalación de dependencias y configuración de variables de entorno.
```

## Database Modeling

```
Revisa el modelo de datos en ModeloDatos.md y sugiere mejoras u optimizaciones basadas en las buenas prácticas de DDD mencionadas en ManifestoBuenasPracticas.md
```

## Docker and PostgreSQL Setup

```
Proporciona instrucciones paso a paso para configurar y ejecutar la base de datos PostgreSQL usando Docker para este proyecto, incluyendo cómo aplicar las migraciones y seeds iniciales.
```

## SOLID and DRY Implementation

```
Identifica áreas en el código backend donde se podrían aplicar mejor los principios SOLID y DRY basándote en las recomendaciones del ManifestoBuenasPracticas.md
```

## Frontend-Backend Integration

```
Explica cómo implementar correctamente una nueva funcionalidad que requiera cambios tanto en el frontend como en el backend, siguiendo las mejores prácticas del proyecto.
```

## GitHub Actions Pipeline Setup

```
Realiza el ejercicio:
Tu misión en este ejercicio es crear un pipeline en GitHub Actions que, tras el trigger "push a una rama con un Pull Request abierto", siga los siguientes pasos:
1. Pase unos tests de backend.
2. Genere un build del backend.
3. Despliegue el backend en un EC2. 
Para ello, debes seguir estos pasos:
- Configurar el workflow de GitHub Actions en un archivo .github/workflows/pipeline.yml.
- Documentar los prompts utilizados para generar cada paso del pipeline:
  - Tests de backend.
  - Generación del build del backend.
  - Despliegue del backend en EC2.
- Asegúrate de que el pipeline se dispare con un push a una rama con un Pull Request abierto.
```

### Implementación del Pipeline con GitHub Actions

Para crear este pipeline, he utilizado los siguientes prompts específicos para cada paso:

#### 1. Paso de Tests de Backend

```
Crea un job en GitHub Actions que ejecute los tests del backend de mi aplicación Node.js/TypeScript.
- La aplicación usa Jest para las pruebas
- Necesita Node.js 16
- Los tests están en el directorio backend/
- Quiero que se ejecute solo cuando hay un push a una rama con un PR abierto
```

#### 2. Generación del Build del Backend

```
Crea un job en GitHub Actions que genere el build del backend TypeScript.
- Usa Node.js 16
- El comando de build es "npm run build" dentro del directorio backend/
- Debe ejecutarse solo si los tests pasaron con éxito
- Debe guardar los archivos compilados como artefactos para usar en el despliegue
```

#### 3. Despliegue en EC2

```
Crea un job en GitHub Actions para desplegar mi aplicación backend a una instancia EC2.
- Necesito configurar credenciales de AWS
- Debe establecer una conexión SSH a la instancia
- Debe transferir los archivos de build, package.json y el directorio prisma
- Debe instalar dependencias de producción
- Debe generar los clientes Prisma
- Debe reiniciar la aplicación con PM2
- Debe ejecutarse solo si el build fue exitoso y hay un PR abierto
```

### Código Completo del Pipeline

El pipeline completo está configurado en el archivo `.github/workflows/pipeline.yml` con el siguiente contenido:

```yaml
name: Backend CI/CD Pipeline

on:
  push:
    branches:
      - '*'
    paths:
      - 'backend/**'
      - '.github/workflows/**'
  pull_request:
    types: [opened, synchronize, reopened]
    paths:
      - 'backend/**'
      - '.github/workflows/**'

jobs:
  test:
    name: Test Backend
    runs-on: ubuntu-latest
    # Se ejecuta cuando hay un push a una rama con un PR abierto
    if: github.event_name == 'push' && github.event.pull_request

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '16'
          cache: 'npm'
          cache-dependency-path: 'backend/package-lock.json'

      - name: Install dependencies
        run: cd backend && npm install

      - name: Run tests
        run: cd backend && npm test
        env:
          DATABASE_URL: postgresql://postgres:password@localhost:5432/test_db

  build:
    name: Build Backend
    runs-on: ubuntu-latest
    needs: test
    if: success()

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '16'
          cache: 'npm'
          cache-dependency-path: 'backend/package-lock.json'

      - name: Install dependencies
        run: cd backend && npm install

      - name: Build backend
        run: cd backend && npm run build

      - name: Upload build artifacts
        uses: actions/upload-artifact@v3
        with:
          name: build
          path: backend/dist

  deploy:
    name: Deploy to EC2
    runs-on: ubuntu-latest
    needs: build
    if: success() && github.event_name == 'push' && github.event.pull_request

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Download build artifacts
        uses: actions/download-artifact@v3
        with:
          name: build
          path: backend/dist

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_ID }}
          aws-secret-access-key: ${{ secrets.AWS_ACCESS_KEY }}
          aws-region: us-east-1

      - name: Setup SSH
        uses: webfactory/ssh-agent@v0.7.0
        with:
          ssh-private-key: ${{ secrets.SSH_PRIVATE_KEY }}

      - name: Deploy to EC2
        env:
          EC2_INSTANCE: ${{ secrets.EC2_INSTANCE }}
          SSH_USER: ec2-user
        run: |
          # Asegurar que el directorio de destino existe
          ssh -o StrictHostKeyChecking=no $SSH_USER@$EC2_INSTANCE "mkdir -p ~/app/backend"
          
          # Transferir archivos al servidor
          scp -o StrictHostKeyChecking=no -r backend/dist/* backend/package.json backend/package-lock.json backend/prisma $SSH_USER@$EC2_INSTANCE:~/app/backend/
          
          # Instalar dependencias y configurar la base de datos
          ssh -o StrictHostKeyChecking=no $SSH_USER@$EC2_INSTANCE "cd ~/app/backend && npm install --production && npx prisma generate"
          
          # Configurar variables de entorno si no existen
          ssh -o StrictHostKeyChecking=no $SSH_USER@$EC2_INSTANCE "cd ~/app/backend && [ -f .env ] || echo 'DATABASE_URL=${{ secrets.DATABASE_URL }}' > .env"
          
          # Reiniciar o iniciar la aplicación con PM2
          ssh -o StrictHostKeyChecking=no $SSH_USER@$EC2_INSTANCE "cd ~/app/backend && pm2 restart backend || pm2 start dist/index.js --name backend"
```

### Secretos Necesarios

Para que el pipeline funcione correctamente, deben configurarse los siguientes secretos en el repositorio:

1. `AWS_ACCESS_ID`: El ID de acceso de AWS
2. `AWS_ACCESS_KEY`: La clave secreta de acceso de AWS
3. `EC2_INSTANCE`: La dirección IP o DNS de la instancia EC2
4. `SSH_PRIVATE_KEY`: La clave SSH privada para conectar a la instancia EC2
5. `DATABASE_URL`: La URL de conexión a la base de datos PostgreSQL

Este pipeline completo automatiza el proceso de prueba, compilación y despliegue del backend, asegurando que solo se despliegue código que haya pasado todas las pruebas.