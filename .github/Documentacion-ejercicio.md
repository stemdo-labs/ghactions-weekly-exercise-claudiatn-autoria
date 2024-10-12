## 1. Creación de Entornos

Primero, configuré dos entornos diferentes en el proyecto, Production y UAT, desde Settings > Environment. 
Para poder crear estos entornos, fue necesario cambiar el repositorio a público.

![image](https://github.com/user-attachments/assets/76b1d068-b7a3-4d78-94fc-0096852968d2)

![image](https://github.com/user-attachments/assets/fbb20e1d-f306-4155-85eb-665fa9126fee)

## 2. Configuración de Variables y Secretos
En cada entorno (Production y UAT), se crearon los siguientes elementos para facilitar la conexión con DockerHub:

  - DOCKER_PASSWORD: Un secreto que almacena la contraseña de DockerHub.
  - DOCKER_USERNAME: Una variable que contiene el nombre de usuario de DockerHub.

Estas variables y secretos serán utilizados más adelante para autenticar la conexión a DockerHub.

![image](https://github.com/user-attachments/assets/d3da434d-97b2-4372-9aa1-8e6ee8ddae19)


## 4. Creación de un Action para generar el tag de la imagen

Se creó un Action que extrae tanto el nombre de la aplicación como la versión desde el archivo package.json. 
Este action busca estos valores en el archivo para generar el nombre completo de la aplicación en el siguiente formato:

          <nombre-aplicación>:<versión>


Este nombre generado será el que se utiliza más adelante para usar como tag para imagen que crearé y subiré a DockerHub. 
El action se encuentra en .github/actions/taggear_image.


          name: "Taggear Imagen de docker"
          
          outputs:
            tagged:
              description: "Nombre completo para la imagen taggeada"
              value: "${{steps.taggear.outputs.tagged}}"
          
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
          
              - name: "Extraer nombre de la app"
                id: app_name
                shell: bash
                run: |
                  app_name=$(jq -r '.name' ./package.json)
                  echo "App name: $app_name"
                  echo "App_name=${app_name}" >> $GITHUB_OUTPUT
                  
              - name: "Nombre completo Imagen"
                id: taggear
                shell: bash
                run: |
                  tagged_image="${{  steps.app_name.outputs.app_name }}:${{ steps.version.outputs.version}}"
                  echo "Tagged image: $tagged_image"
                  echo "tagged=${tagged_image}" >> $GITHUB_OUTPUT
            

## 5. Creación de un Workflow principal para generar todo el flujo CI/CD

Luego dentro de mi carpeta workflow se pueden encontrar tres archivos: 

  - workflow-main.yaml
  - ci-build-push-docker-image.yaml
  - cd-deply-verify.yaml


 ## Workflow-main 

Este worflow se va aajecutar cuando se realice un push en las ramas main o development. Y contiene tres trabajos princiaples:

- Entorno:
    Su objetivo es definir el entorno. Para ello dependiendo de la rama en la que este main o development se le asigna un valor a la
    valiable environment de Production o UAT (Son los entornos anteriormente configurados en el repositorio).
      
        -  main => production
        - development => UAT

- Call-workflow-ci:
     Este trabajo usa un workflow reusabe almacenado en .github/workflows/ci-build-push-docker-image.yaml.
    y que explicaré mas adelante. Tendrá un needs del trabajo entorno para asegurarse de que el entorno ya ha sido definido y además se le pasará
    como entrada ese entorno. Por último se necesita establecer secrets_inhert para que herede los secretos definidios anteriormente.


-Call-workflow-cd:
    Este trabajo usa un workflow reusabe almacenado en .github/workflows/cd-deploy-verify.yaml. 
    Además este dependerá de los trabajos de entorno y de call-workflow-ci para asegurarse de que tanto el entorno como el proceso de CI han sido
    ejecutados correctamente. En este caso se le pasarán dos entradas, el environment como ene l anterior trabajo y el tag que será el nombre de la imagen  
    definida en el workflow de ci.
    También necesita secrets_inhert para que herede los secretos definidios anteriormente.


        name: Workflow principal
        on:
          push:
            branches:
              - main
              - development
        
        jobs:
          entorno:
            runs-on: ubuntu-latest
            outputs:
              environment: ${{ steps.set-env.outputs.environment }} 
            steps:
              - name: Imprimir rama del push
                run: echo "El push se realizó en la rama ${{ github.ref_name }}"
                
              - name: Definir entorno
                id: set-env
                run: |
                  if [[ "${{ github.ref_name }}" == "main" ]]; then
                    echo "environment=production" >> $GITHUB_OUTPUT
                  else
                    echo "environment=uat" >> $GITHUB_OUTPUT
                  fi
              
              - name: Imprimir entorno
                run: echo "El entorno es  ${{ steps.set-env.outputs.environment }}"
        
              
          call-workflow-ci:
            needs: [entorno]
            uses: ./.github/workflows/ci-build-push-docker-image.yaml
            with:
              environment: ${{ needs.entorno.outputs.environment }}
            secrets: inherit 
           
          call-workflow-cd:
            needs: [entorno,call-workflow-ci]
            uses: ./.github/workflows/cd-deply-verify.yaml
            with:
              environment: ${{ needs.entorno.outputs.environment }}
              tag: ${{ needs.call-workflow-ci.outputs.tag }}
            secrets: inherit 

      
 ## 6. Workflow CI

  Este workflow de integración continua realiza varias tareas para la construcción de la apllicación, creacción de una imagen Docker y subirla
  a DockerHub.

  Al ser un workflow reusable se activara cuando otro lo llame (workflow_call) y ademas recipe un input environment indicando el entorno en el que se
  encuentra y un output del tag de la imagen que se genera en su job build

  El primer job que se encuentra definidio es test-app que solo se ejecutará si el entorno es producción.Su objetivo es simular la ejecución de pruebas automatizadas en la aplicación, 
  asegurando que todo funcione correctamente antes de  construir de la imagen Docker.

  El siguiente job es build que se encarga de la construcción de la apllicación, creacción de una imagen Docker y subirla
  a DockerHub. Siempre se ejecuta aunque si se ejecuta test-app como en el caso de produccion espera a que haya finalizado.
  
  Establece una variable global DOCKER_USERNAME que esta definida en mi variable de entronos anteriormente configurada y además establece una salida
  que será el nombre del tag de la imagen de Docker generada.

   Sus pasos son los siguiente:
    - Hace un checkout del codigo para poder usar la action.
    - LLama a la acción personalizada para crear el tag que usará la imagen de Docker.
    - Hace login en DockerHub usando la acción  docker/login-action@v3 y utilizando el nombre de usuario anteriormente establecido en la variable global y para 
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
                    value: ${{ jobs.build.outputs.image_name }}
            
              
            jobs:
              tests-app:
                if: ${{ inputs.environment == 'production' }}
                runs-on: ubuntu-latest
                steps:
                  - name: Ejecutar simulación de test
                    run: | 
                      sleep 20
                      echo "Ejecutando simulación de test en producción"
                    
                  - name: Ejecutar cobertura de código
                    run: | 
                      sleep 20
                      echo "Ejecutando cobertura de código en producción"
                      
              build:
                if: ${{ always() }} 
                needs: [tests-app] 
                runs-on: ubuntu-latest
                environment: ${{inputs.environment}}
                env:
                  DOCKER_USERNAME: ${{ vars.DOCKER_USERNAME }}
                  
                outputs:
                  image_name: ${{ steps.taggear_image.outputs.tagged }}
                  
                  
                steps:
                - name: Checkout
                  uses: actions/checkout@v3
                
                - name: Action tag
                  id: taggear_image
                  uses: ./.github/actions/taggear_image
              
                - name: Nombre del tag
                  run: |
                    echo image_name="${{ steps.taggear_image.outputs.tagged }}" >> $GITHUB_OUTPUT
            
                   
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
                    docker build -t $DOCKER_USERNAME/${{ steps.taggear_image.outputs.tagged }} .
                    docker images
                    
            
                - name: Subir Docker image
                  run: |
                    docker push $DOCKER_USERNAME/${{ steps.taggear_image.outputs.tagged }}

 
    
## 7. Workflow CD

Este workflow se encarga del despliegue continuo, el cual descargará una imagen Docker de DockerHub y lo desplegará en un entorno de producción.
Va a recibir dos entradas, el entorno para saber donde tiene que realizar el despligue y el tag para saber cual es l aimagen de Docker que necesita descargar.
Se compone de un job deploy que establece los siguientes pasos:
 - Hace un pull a Docker de la imagen que desea desplegar
 - Inicia el contenerde en segundo plano y exponiendolo en el puerto 8080 como esta configurado en el proyecto de angular.
 - Verifica que el contenedor este corriendo.
 - Verifica que la aplicación esta desplegada haciendo un sleep de 40 para que la aplicación se encuentre lista y usando un curl para hacer una petición a
   localhost:8080- 
      
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

 ## 8. Aprobadores de entorno:

Configuro los reviewers para el ambiente de production, me añado como revisora.

![image](https://github.com/user-attachments/assets/7bbb9487-2244-4730-a965-7beb4f9c4a50)

        
  ## 9.Pruebas:

  ## 9.1 Desplegar en Production:
Cambio la versión manualmente de mi package.json a version: 1.0.0 y al hacer commit automaticamente generará el push.

Comprobamos que el entorno que nos esta cogiendo es el de production ya que lo hemos ejecutado desde la rama main:

![image](https://github.com/user-attachments/assets/b846834a-ac7f-4f3d-9c5d-3169d971bd95)

Se ejecutan los test antes de empezar con el workflow del CI

![image](https://github.com/user-attachments/assets/c6d7f0e8-59e9-48a0-937c-a6c716aab4d1)

Una vez pasado los tests con éxito al estar en un entorno production el worflow se va a quedar esperando la aprobación:

![image](https://github.com/user-attachments/assets/fc83906c-0cd4-485f-936a-ba7d2e1f7d66)

Se ejecuta la aprobación:

![image](https://github.com/user-attachments/assets/66c99962-d1c8-4e20-a316-fb5a0f23f12c)

Y empieza con la ejecucción del workflow ci

![image](https://github.com/user-attachments/assets/08dd7167-1279-4130-99c3-ff148ceba265)

Se puede observar como se van ejecutando los pasos generando el tag de la imagen , haciendo login y subiendo la imagen.

![image](https://github.com/user-attachments/assets/2c3cd863-6593-4479-8c4a-abe029e8f6b5)

Desde DockerHub se puede verificar si la iamgen ha sido subida correctamente.

![image](https://github.com/user-attachments/assets/9e07bc81-7225-43f4-83b8-4ca8cfb23930)

Nuevamente se queda esperando a que aprueben el despliegue.

![image](https://github.com/user-attachments/assets/b114f611-0391-4354-a085-50f2be610daf)


Se aprueba y podemos observar como se han ejecutado correctamente los pasos para desplegar la iamgen en produccion.

![image](https://github.com/user-attachments/assets/c6e9af9b-2c01-45da-8d85-1782890e3307)


El flujo ha sido correctamente ejecutado:

![image](https://github.com/user-attachments/assets/669ec6ab-a02b-4cf0-84cf-a8edd128cae0)



  ## 9.2 Desplegar en UAT:

Ahora desde el entorno de UAT cambio la versión manualmente de mi package.json a version: 1.0.1 y al hacer el push comenzará el flujo.

![image](https://github.com/user-attachments/assets/771113df-3385-4074-91b2-1210981829ef)

Poder observar en que entorno estamos:

![image](https://github.com/user-attachments/assets/05df02f9-0f74-4002-93b3-4fb64e773ea2)

Al estar en UAt no se tienen que ejecutar los tests y además no necesita de aprobadores ya que no ha sido configurado para este entorno

![image](https://github.com/user-attachments/assets/bebaf53f-048e-4775-baa6-35afb1dd4a03)

Se van ejecutando los pasos del tag de la imagen construcción y subida con la nueva versión.

![image](https://github.com/user-attachments/assets/e284be4a-0224-4aee-b565-5cb758500c96)

Se comprueba en DockerHub que se ha subido correctamente.

![image](https://github.com/user-attachments/assets/9cebe795-f9a7-4ea4-ba16-e08872a86075)

Se puede observar que el despligue en UAT se ha ejecutado con éxito.

![image](https://github.com/user-attachments/assets/c45a6982-4903-4432-9bc1-645a2f9617f8)


Este es el flujo que seguiria un desplieuge en UAT
![image](https://github.com/user-attachments/assets/a441b55d-c17a-486a-b4f8-07318d0d787b)




