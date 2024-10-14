Modifico mi action taggear_image le declaro un input que será entorno para después poder añadir la etiqueta si el entorno es uat:

          name: "Taggear Imagen de docker"
          inputs:
            environment:
              type: string
              required: true
              
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
                   echo Entrono: "${{ inputs.environment }}"
                   if [[ "${{ inputs.environment }}" == "uat" ]]; then
                     tagged_image="${{  steps.name.outputs.name }}:${{ steps.version.outputs.version}}-snapshot"
                      echo "Tagged image: $tagged_image"
                      echo "tagged=${tagged_image}" >> $GITHUB_OUTPUT
                   else
                     tagged_image="${{  steps.name.outputs.name }}:${{ steps.version.outputs.version}}"
                      echo "Tagged image: $tagged_image"
                      echo "tagged=${tagged_image}" >> $GITHUB_OUTPUT
                   fi


Desde mi workflow ci se le pasara este entorno cuando se llama a la action:

            name: CI
            on:
              workflow_dispatch:
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
                     with:
                       environment:  ${{ inputs.environment }}
                       
            
                   - name: Imprimir tag
                     run: |
                       echo tag_image: ${{ steps.tag.outputs.tagged }}
              
              build:
                if: ${{ always() }} 
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
            
             
    
Desplegamos en main que será el entorno de production y se peude ver que no se ha añadido la etiqueta.

![image](https://github.com/user-attachments/assets/30ff7d33-d1b0-4bc0-a742-4dd7556f45ac)

![image](https://github.com/user-attachments/assets/b8cc1fef-2686-4ac1-a959-dc8043f8eaf9)


Si desplegamos desde development que será el entorno uat se añade la etiqueta

![image](https://github.com/user-attachments/assets/2672cda6-79cc-47db-9142-613911556cce)

![image](https://github.com/user-attachments/assets/a2687586-86a2-4c5c-9564-c2bda43fd7f1)


Desde DockerHub se puede observar como se han subido las dos imagenes:

![image](https://github.com/user-attachments/assets/bef1b281-f7ca-4e38-86ba-ccea1702b52c)


      
      
