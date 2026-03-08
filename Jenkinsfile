pipeline {

  agent any

  options {
    timestamps()
    ansiColor('xterm')
    timeout(time: 60, unit: 'MINUTES')
    buildDiscarder(logRotator(numToKeepStr: '10'))
    disableConcurrentBuilds()
  }

  parameters {
    string(name: 'TF_ARTIFACT_BUILD', defaultValue: 'latest')
    string(name: 'CHEF_ARTIFACT_BUILD', defaultValue: 'latest')

    choice(
      name: 'ENVIRONMENT',
      choices: ['dev','staging','prod']
    )

    booleanParam(name: 'APPLY_TERRAFORM', defaultValue: true)
    booleanParam(name: 'RUN_CHEF', defaultValue: true)
    booleanParam(name: 'DESTROY', defaultValue: false)
  }

  environment {

    TF_PATH = "C:/tools/terraform"

    NEXUS_URL  = 'http://127.0.0.1:8081'
    TF_REPO    = 'terraform-artifacts'
    CHEF_REPO  = 'chef-artifacts'

    AWS_REGION = 'us-east-1'

    STATE_BUCKET = 'terraform-state-364829013514'
    S3_BUCKET    = 'devops-chef-cookbooks-364829013514'

    SLACK_CHANNEL = '#jenkins'
  }

  stages {

    stage('Clean Workspace') {
      steps {
        cleanWs()
      }
    }

    stage('Pre Flight Check') {
      steps {

        withCredentials([[
          $class: 'AmazonWebServicesCredentialsBinding',
          credentialsId: 'aws-credentials'
        ]]) {

          powershell '''

          Write-Host "====== PIPELINE PREFLIGHT ======"

          aws sts get-caller-identity

          aws s3 ls s3://$env:STATE_BUCKET --region $env:AWS_REGION

          if ($LASTEXITCODE -ne 0) {
            Write-Host "ERROR: Terraform state bucket not accessible"
            exit 1
          }

          Write-Host "Preflight check passed"

          '''
        }
      }
    }

    stage('Download Terraform Artifact') {

      steps {

        withCredentials([
          usernamePassword(
            credentialsId: 'nexus-credentials',
            usernameVariable: 'NEXUS_USER',
            passwordVariable: 'NEXUS_PASS'
          )
        ]) {

          powershell '''

          New-Item terraform -ItemType Directory -Force | Out-Null
          New-Item artifacts -ItemType Directory -Force | Out-Null

          if ($env:TF_ARTIFACT_BUILD -eq "latest") {

            $listUrl = "$env:NEXUS_URL/service/rest/v1/components?repository=$env:TF_REPO"

            curl.exe -s -u "$env:NEXUS_USER`:$env:NEXUS_PASS" -o tf_components.json $listUrl

            $response = Get-Content tf_components.json | ConvertFrom-Json

            $latest = $response.items |
                      Sort-Object { $_.assets[0].lastModified } -Descending |
                      Select-Object -First 1

            $tfFile = $latest.assets[0].path.Split("/")[-1]

          }
          else {
            $tfFile = "terraform-infra-$env:TF_ARTIFACT_BUILD.tar.gz"
          }

          Write-Host "Downloading $tfFile"

          curl.exe -u "$env:NEXUS_USER`:$env:NEXUS_PASS" `
                   -o "artifacts/$tfFile" `
                   "$env:NEXUS_URL/repository/$env:TF_REPO/$tfFile"

          if ($LASTEXITCODE -ne 0) {
            Write-Host "Artifact download failed"
            exit 1
          }

          tar -xzf "artifacts/$tfFile" -C terraform/

          if (!(Test-Path "terraform/main.tf")) {
            Write-Host "ERROR: Terraform artifact corrupted"
            exit 1
          }

          Write-Host "Terraform artifact ready"

          '''
        }
      }
    }

    stage('Terraform Init') {

      steps {

        retry(3) {

          withCredentials([[
            $class: 'AmazonWebServicesCredentialsBinding',
            credentialsId: 'aws-credentials'
          ]]) {

            powershell '''

            $env:PATH="$env:TF_PATH;$env:PATH"

            cd terraform

            terraform init `
              -backend-config="bucket=$env:STATE_BUCKET" `
              -backend-config="key=mywebapp/$env:ENVIRONMENT/terraform.tfstate" `
              -backend-config="region=$env:AWS_REGION" `
              -backend-config="dynamodb_table=terraform-state-lock" `
              -backend-config="encrypt=true"

            '''
          }
        }
      }
    }

    stage('Terraform Validate') {

      steps {

        powershell '''

        $env:PATH="$env:TF_PATH;$env:PATH"

        cd terraform

        terraform fmt -check
        terraform validate

        '''
      }
    }

    stage('Terraform Plan') {

      steps {

          withCredentials([[
              $class: 'AmazonWebServicesCredentialsBinding',
              credentialsId: 'aws-credentials'
            ]]) {

        powershell '''

        $env:PATH="$env:TF_PATH;$env:PATH"

        cd terraform

        if ($env:DESTROY -eq "true") {

          terraform plan -destroy `
            -var-file="$env:ENVIRONMENT.tfvars" `
            -out=tfplan

        }
        else {

          terraform plan `
            -var-file="$env:ENVIRONMENT.tfvars" `
            -out=tfplan
        }

        '''
            }
        archiveArtifacts artifacts: 'terraform/tfplan'
            }
    }

    stage('Approval Gate') {

      when {
        expression { params.ENVIRONMENT == 'prod' || params.DESTROY }
      }

      steps {

        input message: "Approve infrastructure change?",
              ok: "Proceed"
      }
    }

    stage('Terraform Apply') {

      when {
        expression { params.APPLY_TERRAFORM }
      }

      steps {

        retry(2) {

          withCredentials([[
              $class: 'AmazonWebServicesCredentialsBinding',
              credentialsId: 'aws-credentials'
            ]]) {

          powershell '''

          $env:PATH="$env:TF_PATH;$env:PATH"

          cd terraform

          terraform apply -auto-approve tfplan

          terraform output -json > ../tf_outputs.json

          '''
            }
        }
      }
    }

    stage('Upload Chef Cookbooks') {

      when {
        expression { params.RUN_CHEF && !params.DESTROY }
      }

      steps {

        powershell '''

        aws s3 cp chef/cookbooks `
          s3://$env:S3_BUCKET/chef/cookbooks `
          --recursive `
          --region $env:AWS_REGION

        '''
      }
    }

    stage('Run Chef via SSM') {

      when {
        expression { params.RUN_CHEF && !params.DESTROY }
      }

      steps {

        powershell '''

        $outputs = Get-Content tf_outputs.json | ConvertFrom-Json
        $ec2Id = $outputs.ec2_instance_id.value

        aws ssm send-command `
          --document-name "AWS-RunPowerShellScript" `
          --instance-ids $ec2Id `
          --parameters commands=["chef-client --local-mode --runlist 'recipe[webserver]' --chef-license accept"] `
          --timeout-seconds 600 `
          --region $env:AWS_REGION

        '''
      }
    }

    stage('Verify Deployment') {

      when {
        expression { params.APPLY_TERRAFORM && !params.DESTROY }
      }

      steps {

        powershell '''

        $outputs = Get-Content tf_outputs.json | ConvertFrom-Json

        '''
      }
    }

  }

   post {
    success {
      slackSend(
        channel: env.SLACK_CHANNEL,
        color: 'good',
        message: """
  :white_check_mark: *BUILD SUCCESS* - Terraform
  *Job:* ${env.JOB_NAME}
  *Build:* #${env.BUILD_NUMBER}
  *Duration:* ${currentBuild.durationString}
  *Artifact:* terraform-infra-${env.BUILD_NUMBER}.tar.gz pushed to Nexus
  *URL:* ${env.BUILD_URL}
        """.trim()
      )
    }
    failure {
      slackSend(
        channel: env.SLACK_CHANNEL,
        color: 'danger',
        message: """
  :x: *BUILD FAILED* - Terraform Deploy
  *Job:* ${env.JOB_NAME}
  *Build:* #${env.BUILD_NUMBER}
  *Duration:* ${currentBuild.durationString}
  *URL:* ${env.BUILD_URL}console
        """.trim()
      )
    }
    unstable {
      slackSend(
        channel: env.SLACK_CHANNEL,
        color: 'warning',
        message: """
  :warning: *BUILD UNSTABLE* - Terraform Deploy
  *Job:* ${env.JOB_NAME}
  *Build:* #${env.BUILD_NUMBER}
  *URL:* ${env.BUILD_URL}
        """.trim()
      )
    }
    always {
      cleanWs()
    }
  }
}