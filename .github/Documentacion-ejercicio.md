## 1. Creación de Entornos

Primero, se configuran dos entornos diferentes en el proyecto, Production y UAT, desde Settings > Environment. 
Para poder crear estos entornos es necesario cambiar el repositorio a público.

![image](https://github.com/user-attachments/assets/76b1d068-b7a3-4d78-94fc-0096852968d2)

![image](https://github.com/user-attachments/assets/fbb20e1d-f306-4155-85eb-665fa9126fee)

## 2. Configuración de Variables y Secretos
En cada entorno (Production y UAT), se crean los siguientes elementos para facilitar la conexión con DockerHub:

  - DOCKER_PASSWORD: Un secreto que almacena la contraseña de DockerHub.
  - DOCKER_USERNAME: Una variable que contiene el nombre de usuario de DockerHub.

Estas variables y secretos serán utilizados más adelante para autenticar la conexión a DockerHub.

![image](https://github.com/user-attachments/assets/d3da434d-97b2-4372-9aa1-8e6ee8ddae19)

## 3. Creación de un Action para extraer la versión

Este action extrae la versión de la aplicación en el archivo package.json y la devuelve en una salida (output). La ruta del action es .github/actions/version

        name: "Extraer versión"
        
        outputs:
          version:
            description: "Versión de la app"
            value: "${{steps.version.outputs.version}}"
            
        runs:
          using: "composite"
          steps:
            - name: "Extraer versión"
              id: version
              shell: bash
              run: |
                version=$(jq -r '.version' ./package.json)
                echo "Image version: $version"
                echo  "version=${version}" >> $GITHUB_OUTPUT


## 4. Creación de un Action para extraer el nombre de la aplicación

Este action extrae el nombre de la aplicación en el archivo package.json y la devuelve en una salida (output). La ruta del action es .github/actions/name-app

        name: Nombre de la app
        
        outputs:
          name:
            description: "Nombre de apliacion"
            value: "${{steps.name.outputs.app_name}}"
        
        runs:
          using: "composite"
          steps:
          - name: "Nombre de la aplición"
            id: name
            shell: bash
            run: |
                app_name=$(jq -r '.name' ./package.json)
                echo "App name: $app_name"
                echo "app_name=${app_name}" >> $GITHUB_OUTPUT

## 5. Creación de un Action para generar el tag de la imagen

Es el Action que hace uso de los actions anteriores ( versión y nombre de la aplicación ) para crear la etiqueta con el siguiente formato:

          <nombre-aplicación>:<versión>

Este nombre generado será el que se utilice más adelante para usar como tag de la imagen de Docker la cual se creará y se subirá a DockerHub. 
El action se encuentra en .github/actions/taggear_image.

        name: "Taggear Imagen de docker"
        
        outputs:
          tagged:
            description: "Nombre completo para la imagen taggeada"
            value: "${{steps.taggear.outputs.tagged}}"
            
        runs:
          using: "composite"
          steps:
            - name: "Version"
              id: version
              uses: ./.github/actions/version
        
            - name: "Nombre app"
              id: name
              uses: ./.github/actions/name-app
                
            - name: "Nombre completo Imagen"
              id: taggear
              shell: bash
              run: |
                tagged_image="${{  steps.name.outputs.name }}:${{ steps.version.outputs.version}}"
                echo "Tagged image: $tagged_image"
                echo "tagged=${tagged_image}" >> $GITHUB_OUTPUT



## 6. Creación de un action para la conficuración del entrono

Su objetivo es definir el entorno. Para ello dependiendo de la rama en la que este, main o development, se le asigna un valor a la
variable environment de Production o UAT (Son los entornos anteriormente configurados en el repositorio).
Este valor se define como una salida (output) que se usará en mi workflow principal.

      
        -  main => production
        - development => UAT


Se encuentra en .github/actions/environment.

      name: "Enviroment"
      
      outputs:
        environment:
          description: "Entorno donde me encuentro"
          value: "${{steps.set-env.outputs.environment}}"
          
      runs:
        using: "composite"
        steps:
          - name: Imprimir rama del push
            shell: bash
            run: echo "El push se realizó en la rama ${{ github.ref_name }}"
              
          - name: Definir entorno
            id: set-env
            shell: bash
            run: |
              if [[ "${{ github.ref_name }}" == "main" ]]; then
                echo "environment=production" >> $GITHUB_OUTPUT
              else
                echo "environment=uat" >> $GITHUB_OUTPUT
              fi
            
          - name: Imprimir entorno
            shell: bash
            run: echo "El entorno es  ${{ steps.set-env.outputs.environment }}"


## 7. Cracción de un Action para los tests de la aplicación:

Este action ejecuta los test de la aplicación. Se encuentra en  .github/actions/tests-app

      name: Tests app
      
      runs:
          using: "composite"
          steps:
                
            - name: Ejecutar simulación de test
              shell: bash
              run: | 
                sleep 20
                echo "Ejecutando simulación de test en producción"
              
            - name: Ejecutar cobertura de código
              shell: bash
              run: | 
                sleep 20
                echo "Ejecutando cobertura de código en producción"


## 8. Creación de un Workflow principal para generar todo el flujo CI/CD

Luego dentro de mi carpeta workflow se encuentran tres archivos: 

  - workflow-main.yaml
  - ci-build-push-docker-image.yaml
  - cd-deply-verify.yaml


 ## Workflow-main 

Este worflow se va a ejecutar cuando se realice un push en las ramas main o development. Este contiene tres trabajos princiaples (jobs):

- Entorno:
    Este job llama usa el action personalizada de environment para definir el entorno y que su valor sea accesible para otros jobs.

- Call-workflow-ci:
    Este trabajo usa un workflow reusabe almacenado en .github/workflows/ci-build-push-docker-image.yaml  que se explicará mas adelante.
    Tiene un needs del trabajo entorno para asegurarse de que el entorno ya ha sido definido y además se le pasará como entrada (input) ese entorno. Por último, se necesita establecer 
    secrets_inhert para que herede los secretos definidios anteriormente.


-Call-workflow-cd:
    Este job usa un workflow reusabe almacenado en .github/workflows/cd-deploy-verify.yaml. 
    Además este dependerá de los jobs de entorno y de call-workflow-ci para asegurarse de que tanto el entorno como el proceso de CI han sido
    ejecutados correctamente. En este caso se le pasarán dos entradas, el environment como en el anterior job y el tag que es el nombre de la imagen  
    definida en el workflow de ci.
    También necesita secrets_inhert para que herede los secretos definidios anteriormente.
    
        name: Workflow principal
        on:
          push:
            branches:
              - main
              - development
          workflow_dispatch:
            
        
        jobs:
          entorno:
            runs-on: ubuntu-latest
            outputs: 
              environment: ${{ steps.environment.outputs.environment }}
              
            steps:
            - name: Checkout
              uses: actions/checkout@v3
            
            - name: Action environment
              id: environment
              uses: ./.github/actions/environment
          
            - name: Entorno
              run: |
                echo environment="${{ steps.environment.outputs.environment }}"
              
          call-workflow-ci:
            needs: [entorno]
            uses: ./.github/workflows/ci-build-push-docker-image.yaml
            with:
              environment: ${{ needs.entorno.outputs.environment }}
            secrets: inherit 
           
          call-workflow-cd:
            needs: [entorno,call-workflow-ci]
            uses: ./.github/workflows/cd-deploy-verify.yaml
            with:
              environment: ${{ needs.entorno.outputs.environment }}
              tag: ${{ needs.call-workflow-ci.outputs.tag }}
            secrets: inherit 
      
 ## 9. Workflow CI

  Este workflow de integración continua.
  
  Al ser un workflow reusable se activa cuando otro lo llama (workflow_call) y además recibe un input environment indicando el entorno en el que se
  encuentra y un output del tag de la imagen que se genera en su job tag_image.
  
  Consta de tres jobs: (No se si es la mejor practica en este caso utilizar tres jobs ya que va a ser una misma máquina y los tres van a necesitar el checkout pero he decidido usar los tres para practicar los outputs       entre jobs)
  
  El primer job que se encuentra definidio es test-app que solo se ejecutará si el entorno es producción. Si está en el entorno de producción se usará el action de test anteriormente definido.
  Su objetivo es simular la ejecución de pruebas automatizadas en la aplicación asegurando que todo funcione correctamente antes de construir de la imagen Docker.

  El segundo job es tag_image que es el encargado de usar el action para definir el tag de la imagen. Este job siempre se ejecuta aunque si se ejecuta test-app como en el caso de produccóon espera a que haya finalizado
  
  El siguiente job, build, es responsable de la creación de una imagen Docker y su posterior subida a DockerHub. Este job se ejecuta únicamente después de que se haya generado el tag de la imagen. Además, utiliza la        variable DOCKER_USERNAME, que ha sido previamente definida en las variables de entorno, para facilitar su uso en todos los pasos del job. También establece una salida, que es el nombre del tag de la imagen Docker     º   generada.

   Sus pasos son los siguiente:
    - Hace un checkout del codigo para poder usar action.
    - Hace login en DockerHub usando la action  docker/login-action@v3 y utilizando el nombre de usuario anteriormente establecido en la variable global y para 
      la contraseña usa el secreto almacenado en secrets.DOCKER_PASSWORD.
    - Construye la imagen de Docker con el tag generado.
    - Sube la imagen a DockerHub.
            
            name: CI
            on:
              workflow_call:
                inputs:
                  environment:
                    type: string
                    required: true
            
                outputs:
                  tag:
                    value: ${{ jobs.tag_image.outputs.tag_image }}
            
              
            jobs:
              tests-app:
                if: ${{ inputs.environment == 'production' }}
                runs-on: ubuntu-latest
                steps:
                  - name: Checkout
                    uses: actions/checkout@v3
                  - name: test app
                    uses: ./.github/actions/tests-app
              
              tag_image:
                 if: ${{ always() }} 
                 needs: tests-app
                 runs-on: ubuntu-latest
                 outputs:
                  tag_image: ${{ steps.tag.outputs.tagged }}
            
                 steps:
                   - name: Checkout
                     uses: actions/checkout@v3
                    
                   - name: Tag de la imagen de Docker
                     id: tag
                     uses: ./.github/actions/taggear_image
            
                   - name: Imprimir tag
                     run: |
                       echo tag_image: ${{ steps.tag.outputs.tagged }}
              
              build:
                needs: tag_image
                runs-on: ubuntu-latest
                environment: ${{inputs.environment}}
                
                env:
                  DOCKER_USERNAME: ${{ vars.DOCKER_USERNAME }}
                  
                  
                steps:
                - name: Checkout
                  uses: actions/checkout@v3
                   
                - name: Imprimir valores de entrada
                  run: |
                        echo "El entorno recibido es: ${{ inputs.environment }}"
                        echo "El nombre de usuario de Docker es: $DOCKER_USERNAME"
            
            
                - name: Hacer login en DockerHub
                  uses: docker/login-action@v3
                  with:
                    username: ${{ vars.DOCKER_USERNAME }}
                    password: ${{ secrets.DOCKER_PASSWORD }}
            
                - name: Build Docker image
                  run: |
                    docker build -t $DOCKER_USERNAME/${{ needs.tag_image.outputs.tag_image }} .
                    docker images
                    
            
                - name: Subir Docker image
                  run: |
                    docker push $DOCKER_USERNAME/${{ needs.tag_image.outputs.tag_image }}
            
             
  
    
## 10. Workflow CD

Este workflow se encarga del despliegue continuo, el cual descarga una imagen Docker de DockerHub y la despliega en un entorno.
Va a recibir dos entradas, el entorno para saber donde tiene que realizar el despligue y el tag para saber cual es la imagen de Docker que necesita descargar.

Se compone de un job deploy que establece los siguientes pasos:
 - Hace un pull a Docker de la imagen que desea desplegar
 - Inicia el contenerde en segundo plano y exponiendolo en el puerto 8080 como esta configurado en el proyecto de angular.
 - Verifica que el contenedor esta corriendo.
 - Verifica que la aplicación esta desplegada haciendo un sleep de 40 para que la aplicación se encuentre lista y usando un curl para hacer una petición a
   localhost:8080.
      
        name: CD
        on:
          workflow_call:
             inputs:
              environment:
                type: string
                required: true
              tag:
                type: string
                required: true
                
        jobs:
          deploy:
             runs-on: ubuntu-latest
             environment: ${{ inputs.environment }} 
             
             env:
              DOCKER_USERNAME: ${{ vars.DOCKER_USERNAME }}
        
             steps:
              - name: Imprimir valores de entrada
                run: |
                    echo "El entorno recibido es: ${{ inputs.environment }}"
                    echo "El nombre de usuario de Docker es: $DOCKER_USERNAME"
                    echo "El tag de la imagen es: ${{ inputs.tag }}"
        
                  
              - name: Hacer pull de la imagen de Docker
                run: |
                  docker pull $DOCKER_USERNAME/${{ inputs.tag }}
                  
              - name: Desplegar
                run: |
                  docker run -d -p 8080:8080  $DOCKER_USERNAME/${{ inputs.tag }}
        
              - name: Verificar si el contenedor está corriendo
                run: docker ps
        
              - name: Verificar respuesta de la app
                run: |
                   sleep 40
                   curl http://localhost:8080


        
 ## 11. Aprobadores de entorno:

Configuro los reviewers para el ambiente de production añadiendome como revisora.

![image](https://github.com/user-attachments/assets/7bbb9487-2244-4730-a965-7beb4f9c4a50)

        
  ## 12.Pruebas:

  ## 12.1 Desplegar en Production:
Cambio la versión manualmente de mi package.json a version: 1.0.0 y al hacer commit automaticamente generará el push.

Comprobamos que el entorno que nos esta cogiendo es el de production ya que lo hemos ejecutado desde la rama main:

![image](https://github.com/user-attachments/assets/b846834a-ac7f-4f3d-9c5d-3169d971bd95)

Se ejecutan los test antes de empezar con el workflow del CI

![image](https://github.com/user-attachments/assets/dcb9edf2-b5c9-4aec-b2cc-a7675638b6a1)

Una vez pasado los tests con éxito al estar en un entorno production el workflow se va a quedar esperando la aprobación.

![image](https://github.com/user-attachments/assets/14cf56e6-08c4-4d3a-850d-67000d0398d0)

Se ejecuta la aprobación.

![image](https://github.com/user-attachments/assets/3678f6f2-7c96-4d12-9d16-aed668f4accc)


Y empieza con la ejecucción del workflow ci

![image](https://github.com/user-attachments/assets/f76d48a7-6bf4-438c-a4c6-25d809a4138b)


Desde DockerHub se puede verificar si la iamgen ha sido subida correctamente.

![image](https://github.com/user-attachments/assets/9e07bc81-7225-43f4-83b8-4ca8cfb23930)

Nuevamente se queda esperando a que aprueben el despliegue.

![image](https://github.com/user-attachments/assets/b9d97a5b-877e-423e-b439-73353b84dbc3)


Se aprueba y podemos observar como se han ejecutado correctamente los pasos para desplegar la imagen en produccion.

![image](https://github.com/user-attachments/assets/18e41b77-ba40-4979-90a6-f427b49fbc4d)

![image](https://github.com/user-attachments/assets/0a8d707e-0354-4cc8-86ce-4e6a8f9eaec2)


El flujo ha sido correctamente ejecutado.

![image](https://github.com/user-attachments/assets/4a224d25-0fd4-495e-b2f3-871af73d2c00)


  ## 12.2 Desplegar en UAT:

Ahora desde el entorno de UAT cambio la versión manualmente de mi package.json a version: 1.0.1 y al hacer el push (en development) comenzará el flujo.

![image](https://github.com/user-attachments/assets/771113df-3385-4074-91b2-1210981829ef)

Se oberseva en que entorno estamos.

![image](https://github.com/user-attachments/assets/cbd396af-0510-48c4-a0d8-7ea96ba69f09)


Al estar en UAt no se tienen que ejecutar los tests y además no necesita de aprobadores ya que no ha sido configurado para este entorno.

[![image](https://github.com/user-attachments/assets/bebaf53f-048e-4775-baa6-35afb1dd4a03)](https://github.com/stemdo-labs/ghactions-weekly-exercise-claudiatn/actions/runs/11316106397)

Se van ejecutando los pasos del tag de la imagen, la construcción y la subida con la nueva versión.

![image](https://github.com/user-attachments/assets/2d462848-0aee-43fa-8842-d402df76ffda)

![image](https://github.com/user-attachments/assets/ea5ed7cb-5fea-47ce-8f0f-626e2240cf06)


Se comprueba en DockerHub que  la imagen se ha subido correctamente.

![image](https://github.com/user-attachments/assets/9cebe795-f9a7-4ea4-ba16-e08872a86075)

Se puede observar que el despligue en UAT se ha ejecutado con éxito.

![image](https://github.com/user-attachments/assets/c1c70eac-492c-4938-bd57-7cfa2618e141)


Este es el flujo que sigue el despliegue en UAT si todo ha sucedido correctamente.

![image](https://github.com/user-attachments/assets/bf0cb347-08f1-47a4-9067-e9b5f9bce791)





