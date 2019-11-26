name: 0.1.$(Rev:r)

trigger:
  - master

resources:
  - repo: self

variables:
  dockerRegistryServiceConnection: "Fabmedical ACR"
  imageRepository: "content-web"
  containerRegistry: "$(containerRegistryName).azurecr.io"
  containerRegistryName: "acr*****"
  dockerfilePath: "$(Build.SourcesDirectory)/Dockerfile"
  tag: "$(Build.BuildNumber)"
  vmImageName: "ubuntu-latest"

stages:
  - stage: Build
    displayName: Build and Push
    jobs:
      - job: Docker
        displayName: Build and Push Docker Image
        pool:
          vmImage: $(vmImageName)
        steps:
          - checkout: self
            fetchDepth: 1

          - task: Docker@2
            displayName: Build and push an image to container registry
            inputs:
              command: buildAndPush
              repository: $(imageRepository)
              dockerfile: $(dockerfilePath)
              containerRegistry: $(dockerRegistryServiceConnection)
              tags: |
                $(tag)
                latest
      - job: Helm
        displayName: Build and Push Helm Chart
        pool:
         vmImage: $(vmImageName)
        steps:
        - checkout: self
          fetchDepth: 1

        - task: HelmInstaller@1
          inputs:
           helmVersionToInstall: '2.15.2'
          displayName: 'Helm Install'

        - task: HelmDeploy@0
          inputs:
           connectionType: 'None'
           command: 'package'
           chartPath: 'charts/web'
           chartVersion: '$(Build.BuildNumber)'
          displayName: 'Helm Package'

        - task: AzureCLI@1
          inputs:
           azureSubscription: 'azurecloud'
           scriptLocation: 'inlineScript'
           inlineScript: |
              set -euo pipefail

              az acr helm push \
                --name $(containerRegistryName) \
                $(Build.ArtifactStagingDirectory)/web-$(Build.BuildNumber).tgz

               failOnStandardError: true
               displayName: 'Helm Push'
  - stage:
    displayName: AKS Deployment
    jobs:
      - deployment: DeployAKS
        displayName: "Deployment to AKS"
        pool:
          vmImage: $(vmImageName)
        environment: "aks"
        strategy:
          runOnce:
            deploy:
              steps:
                - checkout: none

                - task: HelmInstaller@1
                  inputs:
                    helmVersionToInstall: '2.15.2'
                  displayName: "Helm Install"

                - task: HelmDeploy@0
                  inputs:
                    connectionType: "Azure Resource Manager"
                    azureSubscription: "azurecloud"
                    azureResourceGroup: "fabmedical-*****"
                    kubernetesCluster: "fabmedical-*****"
                    command: "init"
                    arguments: "--service-account tiller"
                  displayName: "Helm init"

                - task: AzureCLI@1
                  inputs:
                    azureSubscription: "azurecloud"
                    scriptLocation: "inlineScript"
                    inlineScript: |
                      set -euo pipefail

                      az acr helm repo add --name $(containerRegistryName)

                    failOnStandardError: true
                  displayName: "Helm repo update"

                - task: HelmDeploy@0
                  inputs:
                    connectionType: "Azure Resource Manager"
                    azureSubscription: "azurecloud"
                    azureResourceGroup: "fabmedical-*****"
                    kubernetesCluster: "fabmedical-*****"
                    command: "upgrade"
                    chartType: "Name"
                    chartName: "$(containerRegistryName)/web"
                    releaseName: "web"
                    overrideValues: "image.tag=$(Build.BuildNumber),image.repository=$(containerRegistry)/content-web"
                    recreate: true
                    force: true
                  displayName: "Helm Upgrade"
