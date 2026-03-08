pipeline {
  agent any

  options {
    timestamps()
    timeout(time: 60, unit: 'MINUTES')
    buildDiscarder(logRotator(numToKeepStr: '5'))
    // Prevent concurrent builds - avoids state conflicts
    disableConcurrentBuilds()
  }

  parameters {
    string(name: 'TF_ARTIFACT_BUILD',   defaultValue: 'latest', description: 'Terraform build number or latest')
    string(name: 'CHEF_ARTIFACT_BUILD', defaultValue: 'latest', description: 'Chef build number or latest')
    choice(name: 'ENVIRONMENT',         choices: ['dev', 'staging', 'prod'], description: 'Target environment')
    booleanParam(name: 'APPLY_TERRAFORM', defaultValue: true,  description: 'Run terraform apply?')
    booleanParam(name: 'RUN_CHEF',        defaultValue: true,  description: 'Run Chef configuration?')
    booleanParam(name: 'DESTROY',         defaultValue: false, description: 'Destroy infrastructure?')
  }

  environment {
    NEXUS_URL     = 'http://127.0.0.1:8081'
    TF_REPO       = 'terraform-artifacts'
    CHEF_REPO     = 'chef-artifacts'
    AWS_REGION    = 'us-east-1'
    S3_BUCKET     = 'devops-chef-cookbooks-364829013514'
    STATE_BUCKET  = 'terraform-state-364829013514'
    SLACK_CHANNEL = '#devops-builds'
  }

  stages {
       stage('Pre-Flight Check') {
      steps {
        withCredentials([[
          $class: 'AmazonWebServicesCredentialsBinding',
          credentialsId: 'aws-credentials']]) {
          powershell '''
            Write-Host "=========================================="
            Write-Host "  DEPLOY PIPELINE PRE-FLIGHT CHECK"
            Write-Host "=========================================="
            Write-Host "Environment  : $env:ENVIRONMENT"
            Write-Host "Apply TF     : $env:APPLY_TERRAFORM"
            Write-Host "Run Chef     : $env:RUN_CHEF"
            Write-Host "Destroy      : $env:DESTROY"
            Write-Host "TF Artifact  : $env:TF_ARTIFACT_BUILD"
            Write-Host "Chef Artifact: $env:CHEF_ARTIFACT_BUILD"
            Write-Host "=========================================="

            # Verify AWS connectivity
            Write-Host "Verifying AWS connectivity..."
            $identity = aws sts get-caller-identity --output json | ConvertFrom-Json
            Write-Host "AWS Account: $($identity.Account)"
            Write-Host "AWS User:    $($identity.Arn)"

            # Verify state bucket exists
            Write-Host "Verifying Terraform state bucket..."
            aws s3 ls s3://$env:STATE_BUCKET --region $env:AWS_REGION | Out-Null
            if ($LASTEXITCODE -ne 0) {
              Write-Host "ERROR: State bucket $env:STATE_BUCKET not found"
              exit 1
            }
            Write-Host "State bucket OK: $env:STATE_BUCKET"

            # Verify Nexus connectivity
            Write-Host "Pre-flight checks passed"
          '''
        }
      }
    }

    stage('Download Terraform Artifact from Nexus') {
  steps {
    withCredentials([usernamePassword(
      credentialsId: 'nexus-credentials',
      usernameVariable: 'NEXUS_USER',
      passwordVariable: 'NEXUS_PASS')]) {
      powershell '''
        # Clean up any leftover files from previous builds
        Write-Host "Cleaning workspace..."
        Remove-Item -Force "*.tar.gz"        -ErrorAction SilentlyContinue
        Remove-Item -Force "tf_components.json"  -ErrorAction SilentlyContinue
        Remove-Item -Recurse -Force "terraform"  -ErrorAction SilentlyContinue
        New-Item -ItemType Directory -Force -Path "terraform" | Out-Null

        if ($env:TF_ARTIFACT_BUILD -eq "latest") {
          Write-Host "Fetching latest Terraform artifact from Nexus..."
          $listUrl = "$env:NEXUS_URL/service/rest/v1/components?repository=$env:TF_REPO"
          $jsonFile = "tf_components.json"

          curl.exe -s -u "$env:NEXUS_USER`:$env:NEXUS_PASS" -o $jsonFile $listUrl
          if ($LASTEXITCODE -ne 0) { Write-Host "ERROR: Failed to query Nexus API"; exit 1 }

          $response = Get-Content $jsonFile | ConvertFrom-Json
          if ($response.items.Count -eq 0) { Write-Host "ERROR: No artifacts found"; exit 1 }

          $latest = $response.items | Sort-Object { $_.assets[0].lastModified } -Descending | Select-Object -First 1
          $tfFile = $latest.assets[0].path.Split("/")[-1]
          Remove-Item $jsonFile -Force -ErrorAction SilentlyContinue
        } else {
          $tfFile = "terraform-infra-$env:TF_ARTIFACT_BUILD.tar.gz"
        }

        Write-Host "Downloading: $tfFile"
        curl.exe -s -w "`nHTTP Status: %{http_code}`n" `
          -u "$env:NEXUS_USER`:$env:NEXUS_PASS" `
          -o "terraform/$tfFile" `
          "$env:NEXUS_URL/repository/$env:TF_REPO/$tfFile"

        if ($LASTEXITCODE -ne 0) { Write-Host "ERROR: Download failed"; exit 1 }

        Write-Host "Extracting $tfFile..."
        tar -xzf "terraform/$tfFile" -C terraform/
        if ($LASTEXITCODE -ne 0) { Write-Host "ERROR: Extract failed"; exit 1 }

        Write-Host "Terraform artifact ready:"
        Get-Location
        Get-ChildItem terraform/ | Select-Object Name
      '''
    }
  }
}

    stage('Download Chef Artifact from Nexus') {
  when {
    expression { return params.RUN_CHEF && !params.DESTROY }
  }
  steps {
    withCredentials([usernamePassword(
      credentialsId: 'nexus-credentials',
      usernameVariable: 'NEXUS_USER',
      passwordVariable: 'NEXUS_PASS')]) {
      powershell '''
        # Clean up any leftover files from previous builds
        Write-Host "Cleaning chef workspace..."
        Remove-Item -Force "chef_components.json" -ErrorAction SilentlyContinue
        Remove-Item -Recurse -Force "chef"         -ErrorAction SilentlyContinue
        New-Item -ItemType Directory -Force -Path "chef" | Out-Null

        if ($env:CHEF_ARTIFACT_BUILD -eq "latest") {
          Write-Host "Fetching latest Chef artifact from Nexus..."
          $listUrl = "$env:NEXUS_URL/service/rest/v1/components?repository=$env:CHEF_REPO"
          $jsonFile = "chef_components.json"

          curl.exe -s -u "$env:NEXUS_USER`:$env:NEXUS_PASS" -o $jsonFile $listUrl
          if ($LASTEXITCODE -ne 0) { Write-Host "ERROR: Failed to query Nexus API"; exit 1 }

          $response = Get-Content $jsonFile | ConvertFrom-Json
          if ($response.items.Count -eq 0) { Write-Host "ERROR: No artifacts found"; exit 1 }

          $latest   = $response.items | Sort-Object { $_.assets[0].lastModified } -Descending | Select-Object -First 1
          $chefFile = $latest.assets[0].path.Split("/")[-1]
          Remove-Item $jsonFile -Force -ErrorAction SilentlyContinue
        } else {
          $chefFile = "chef-config-$env:CHEF_ARTIFACT_BUILD.tar.gz"
        }

        Write-Host "Downloading: $chefFile"
        curl.exe -s -w "`nHTTP Status: %{http_code}`n" `
          -u "$env:NEXUS_USER`:$env:NEXUS_PASS" `
          -o "chef/$chefFile" `
          "$env:NEXUS_URL/repository/$env:CHEF_REPO/$chefFile"

        if ($LASTEXITCODE -ne 0) { Write-Host "ERROR: Download failed"; exit 1 }

        Write-Host "Extracting $chefFile..."
        tar -xzf "chef/$chefFile" -C chef/
        if ($LASTEXITCODE -ne 0) { Write-Host "ERROR: Extract failed"; exit 1 }

        Write-Host "Chef artifact ready:"
        Get-ChildItem chef/ | Select-Object Name
      '''
    }
  }
}

    stage('Terraform Init') {
      steps {
        withCredentials([[
          $class: 'AmazonWebServicesCredentialsBinding',
          credentialsId: 'aws-credentials']]) {
          powershell '''
            $env:PATH = "C://tools//terraform;$env:PATH"
            Set-Location terraform

            Write-Host "Running terraform init with S3 backend..."
            terraform init `
              -backend-config="bucket=$env:STATE_BUCKET" `
              -backend-config="key=mywebapp/$env:ENVIRONMENT/terraform.tfstate" `
              -backend-config="region=$env:AWS_REGION" `
              -backend-config="dynamodb_table=terraform-state-lock" `
              -backend-config="encrypt=true"

            if ($LASTEXITCODE -ne 0) {
              Write-Host "ERROR: terraform init failed"
              exit 1
            }

            Write-Host "Terraform init complete"
            Write-Host "State file: s3://$env:STATE_BUCKET/mywebapp/$env:ENVIRONMENT/terraform.tfstate"
          '''
        }
      }
    }

    stage('Check Existing State') {
      steps {
        withCredentials([[
          $class: 'AmazonWebServicesCredentialsBinding',
          credentialsId: 'aws-credentials']]) {
          powershell '''
            Write-Host "Checking existing Terraform state..."
            $stateList = terraform state list 2>$null
            if ($stateList) {
              Write-Host "Existing resources in state:"
              $stateList
              Write-Host "Total resources: $($stateList.Count)"
            } else {
              Write-Host "No existing state found - fresh deployment"
            }
          '''
        }
      }
    }

    stage('Terraform Plan') {
      steps {
        withCredentials([[
          $class: 'AmazonWebServicesCredentialsBinding',
          credentialsId: 'aws-credentials']]) {
          powershell '''
            $env:PATH = "C://tools//terraform;$env:PATH"
            Set-Location terraform

            $vars = @(
              "-var=environment=$env:ENVIRONMENT",
              "-var=vpc_id=vpc-09ba6d5a99230fa69",
              "-var=subnet_ids=[`"subnet-02e7c3a75b13b7a8d`",`"subnet-0bd4f1b1292eb34b1`"]",
              "-var=ec2_subnet_id=subnet-02e7c3a75b13b7a8d",
              "-var=key_pair_name=devops-jenkins",
              "-var=ami_id=ami-0f7a0c94dce9ab456",
              "-var=instance_type=t3.medium",
              "-var=app_name=mywebapp"
            )

            if ($env:DESTROY -eq "true") {
              Write-Host "Planning DESTROY for $env:ENVIRONMENT..."
              $planArgs = @("plan", "-destroy") + $vars + @("-out=tfplan")
            } else {
              Write-Host "Planning APPLY for $env:ENVIRONMENT..."
              $planArgs = @("plan") + $vars + @("-out=tfplan")
            }

            & terraform $planArgs
            if ($LASTEXITCODE -ne 0) {
              Write-Host "ERROR: terraform plan failed"
              exit 1
            }

            Write-Host "Terraform plan complete"
          '''
        }
      }
    }

    stage('Approval Gate') {
      when {
        expression { return params.ENVIRONMENT == 'prod' || params.DESTROY == true }
      }
      steps {
        script {
          def msg = params.DESTROY ?
            "CONFIRM DESTROY of ${params.ENVIRONMENT} infrastructure?" :
            "Approve deployment to PRODUCTION?"
          input message: msg,
                ok: params.DESTROY ? "Yes Destroy" : "Deploy to Prod",
                submitter: "admin"
        }
      }
    }

    stage('Terraform Apply') {
      when {
        expression { return params.APPLY_TERRAFORM }
      }
      steps {
        withCredentials([[
          $class: 'AmazonWebServicesCredentialsBinding',
          credentialsId: 'aws-credentials']]) {
          powershell '''
            $env:PATH = "C://tools//terraform;$env:PATH"
            Set-Location terraform

            if ($env:DESTROY -eq "true") {
              Write-Host "DESTROYING infrastructure for $env:ENVIRONMENT..."
              terraform apply -auto-approve tfplan
              if ($LASTEXITCODE -ne 0) {
                Write-Host "ERROR: terraform destroy failed"
                exit 1
              }
              Write-Host "Infrastructure destroyed - state updated in S3"

            } else {
              Write-Host "APPLYING infrastructure for $env:ENVIRONMENT..."
              terraform apply -auto-approve tfplan
              if ($LASTEXITCODE -ne 0) {
                Write-Host "ERROR: terraform apply failed"
                exit 1
              }

              # Save outputs for downstream stages
              terraform output -json | Out-File -FilePath "../tf_outputs.json" -Encoding UTF8
              Write-Host "Terraform apply complete - state saved to S3"
              Write-Host "Outputs:"
              Get-Content "../tf_outputs.json"
            }
          '''
        }
      }
    }

    stage('Wait for EC2 Bootstrap') {
      when {
        expression { return params.APPLY_TERRAFORM && !params.DESTROY }
      }
      steps {
        withCredentials([[
          $class: 'AmazonWebServicesCredentialsBinding',
          credentialsId: 'aws-credentials']]) {
          powershell '''
            $outputs = Get-Content "tf_outputs.json" | ConvertFrom-Json
            $ec2Id   = $outputs.ec2_instance_id.value
            Write-Host "Waiting for EC2 $ec2Id to pass status checks..."

            aws ec2 wait instance-status-ok `
              --instance-ids $ec2Id `
              --region $env:AWS_REGION

            if ($LASTEXITCODE -ne 0) {
              Write-Host "ERROR: EC2 did not pass status checks"
              exit 1
            }

            Write-Host "EC2 healthy - waiting 90s for bootstrap scripts to complete..."
            Start-Sleep -Seconds 90
            Write-Host "Ready for Chef configuration"
          '''
        }
      }
    }

    stage('Upload Chef Cookbooks to S3') {
      when {
        expression { return params.RUN_CHEF && !params.DESTROY }
      }
      steps {
        withCredentials([[
          $class: 'AmazonWebServicesCredentialsBinding',
          credentialsId: 'aws-credentials']]) {
          powershell '''
            Write-Host "Uploading Chef cookbooks to S3..."
            aws s3 cp chef/cookbooks `
              s3://$env:S3_BUCKET/chef/cookbooks `
              --recursive `
              --region $env:AWS_REGION

            if ($LASTEXITCODE -ne 0) {
              Write-Host "ERROR: Failed to upload cookbooks to S3"
              exit 1
            }
            Write-Host "Cookbooks uploaded successfully"
          '''
        }
      }
    }

    stage('Run Chef via SSM') {
      when {
        expression { return params.RUN_CHEF && !params.DESTROY }
      }
      steps {
        withCredentials([[
          $class: 'AmazonWebServicesCredentialsBinding',
          credentialsId: 'aws-credentials']]) {
          powershell '''
            $outputs = Get-Content "tf_outputs.json" | ConvertFrom-Json
            $ec2Id   = $outputs.ec2_instance_id.value
            Write-Host "Running Chef on EC2: $ec2Id"

            $commandId = aws ssm send-command `
              --document-name "AWS-RunPowerShellScript" `
              --instance-ids $ec2Id `
              --parameters commands=["
                Write-Host 'Downloading cookbooks from S3...';
                aws s3 cp s3://$env:S3_BUCKET/chef/cookbooks C:/chef/cookbooks --recursive --region $env:AWS_REGION;
                if (`$LASTEXITCODE -ne 0) { exit 1 };
                Write-Host 'Running Chef client...';
                chef-client --local-mode --runlist 'recipe[webserver]' --chef-license accept;
                if (`$LASTEXITCODE -ne 0) { exit 1 };
                Write-Host 'Chef run complete'
              "] `
              --timeout-seconds 600 `
              --region $env:AWS_REGION `
              --output text `
              --query "Command.CommandId"

            if ($LASTEXITCODE -ne 0) {
              Write-Host "ERROR: Failed to send SSM command"
              exit 1
            }

            Write-Host "SSM Command ID: $commandId"
            Write-Host "Polling for completion..."

            $timeout  = 600
            $elapsed  = 0
            $interval = 15

            while ($elapsed -lt $timeout) {
              Start-Sleep -Seconds $interval
              $elapsed += $interval

              $status = aws ssm get-command-invocation `
                --command-id $commandId `
                --instance-id $ec2Id `
                --region $env:AWS_REGION `
                --query "Status" `
                --output text

              Write-Host "[$elapsed s] Status: $status"

              if ($status -eq "Success") {
                Write-Host "Chef run completed successfully"
                aws ssm get-command-invocation `
                  --command-id $commandId `
                  --instance-id $ec2Id `
                  --region $env:AWS_REGION `
                  --query "StandardOutputContent" `
                  --output text
                break

              } elseif ($status -eq "Failed" -or $status -eq "Cancelled" -or $status -eq "TimedOut") {
                Write-Host "ERROR: Chef run failed - Status: $status"
                aws ssm get-command-invocation `
                  --command-id $commandId `
                  --instance-id $ec2Id `
                  --region $env:AWS_REGION `
                  --query "StandardErrorContent" `
                  --output text
                exit 1
              }
            }

            if ($elapsed -ge $timeout) {
              Write-Host "ERROR: Timed out waiting for Chef"
              exit 1
            }
          '''
        }
      }
    }

    stage('Verify Deployment') {
      when {
        expression { return params.APPLY_TERRAFORM && !params.DESTROY }
      }
      steps {
        powershell '''
          $outputs = Get-Content "tf_outputs.json" | ConvertFrom-Json
          $albDns  = $outputs.alb_dns_name.value
          $ec2Ip   = $outputs.ec2_public_ip.value

          Write-Host "=========================================="
          Write-Host "  DEPLOYMENT SUMMARY"
          Write-Host "=========================================="
          Write-Host "ALB DNS  : http://$albDns"
          Write-Host "EC2 IP   : $ec2Ip"
          Write-Host "Health   : http://$albDns/health"
          Write-Host "=========================================="

          Start-Sleep -Seconds 30

          $success = $false
          for ($i = 1; $i -le 5; $i++) {
            try {
              $response = Invoke-WebRequest `
                -Uri "http://$albDns/health" `
                -UseBasicParsing `
                -TimeoutSec 15
              Write-Host "Attempt $i`: HTTP $($response.StatusCode)"
              if ($response.StatusCode -eq 200) {
                Write-Host "SUCCESS: Application healthy at http://$albDns"
                $success = $true
                break
              }
            } catch {
              Write-Host "Attempt $i`: $($_.Exception.Message)"
            }
            Start-Sleep -Seconds 20
          }

          if (-not $success) {
            Write-Host "WARN: Health check did not return 200"
            Write-Host "Check EC2 and IIS manually at http://$ec2Ip"
          }
        '''
      }
    }

  }

  post {
    success {
      script {
        def action = params.DESTROY ? ":boom: *INFRASTRUCTURE DESTROYED*" : ":rocket: *DEPLOYMENT SUCCESS*"
        slackSend(
          channel: env.SLACK_CHANNEL,
          color: params.DESTROY ? 'warning' : 'good',
          message: """
${action}
*Job:* ${env.JOB_NAME}
*Build:* #${env.BUILD_NUMBER}
*Environment:* ${params.ENVIRONMENT}
*Duration:* ${currentBuild.durationString}
*State:* s3://terraform-state-364829013514/mywebapp/${params.ENVIRONMENT}/terraform.tfstate
*URL:* ${env.BUILD_URL}
          """.trim()
        )
      }
    }
    failure {
      slackSend(
        channel: env.SLACK_CHANNEL,
        color: 'danger',
        message: """
:fire: *DEPLOYMENT FAILED*
*Job:* ${env.JOB_NAME}
*Build:* #${env.BUILD_NUMBER}
*Environment:* ${params.ENVIRONMENT}
*Duration:* ${currentBuild.durationString}
*State preserved in:* s3://terraform-state-364829013514/mywebapp/${params.ENVIRONMENT}/terraform.tfstate
*URL:* ${env.BUILD_URL}console
        """.trim()
      )
    }
    always {
      cleanWs()
    }
  }

}