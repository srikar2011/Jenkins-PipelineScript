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
    string(name: 'TF_ARTIFACT_BUILD',   defaultValue: 'latest', description: 'Terraform build number or latest')
    string(name: 'CHEF_ARTIFACT_BUILD', defaultValue: 'latest', description: 'Chef build number or latest')
    choice(name: 'ENVIRONMENT',         choices: ['dev', 'int', 'qa', 'prod'], description: 'Target environment')

    // Select all override
    booleanParam(name: 'SELECT_ALL_MODULES',  defaultValue: false, description: 'Select all modules - overrides individual selections below')

    // Individual module selection
    booleanParam(name: 'DEPLOY_SECURITY',     defaultValue: true,  description: 'Deploy security groups')
    booleanParam(name: 'DEPLOY_NETWORKING',   defaultValue: true,  description: 'Deploy networking')
    booleanParam(name: 'DEPLOY_COMPUTE',      defaultValue: true,  description: 'Deploy EC2 instance')
    booleanParam(name: 'DEPLOY_LOADBALANCER', defaultValue: true,  description: 'Deploy ALB and target group')
    booleanParam(name: 'DEPLOY_DATABASE',     defaultValue: false, description: 'Deploy RDS database')

    booleanParam(name: 'APPLY_TERRAFORM', defaultValue: true,  description: 'Run terraform apply?')
    booleanParam(name: 'RUN_CHEF',        defaultValue: true,  description: 'Run Chef configuration?')
    booleanParam(name: 'DESTROY',         defaultValue: false, description: 'Destroy infrastructure?')
  }

  environment {
    TF_PATH      = 'C:\\tools\\terraform'
    NEXUS_URL    = 'http://127.0.0.1:8081'
    TF_REPO      = 'terraform-artifacts'
    CHEF_REPO    = 'chef-artifacts'
    AWS_REGION   = 'us-east-1'
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
            Write-Host "=========================================="
            Write-Host "  DEPLOY PIPELINE PRE-FLIGHT CHECK"
            Write-Host "=========================================="
            Write-Host "Environment    : $env:ENVIRONMENT"
            Write-Host "Select All     : $env:SELECT_ALL_MODULES"
            Write-Host "Security       : $env:DEPLOY_SECURITY"
            Write-Host "Networking     : $env:DEPLOY_NETWORKING"
            Write-Host "Compute        : $env:DEPLOY_COMPUTE"
            Write-Host "Load Balancer  : $env:DEPLOY_LOADBALANCER"
            Write-Host "Database       : $env:DEPLOY_DATABASE"
            Write-Host "Apply Terraform: $env:APPLY_TERRAFORM"
            Write-Host "Run Chef       : $env:RUN_CHEF"
            Write-Host "Destroy        : $env:DESTROY"
            Write-Host "TF Artifact    : $env:TF_ARTIFACT_BUILD"
            Write-Host "Chef Artifact  : $env:CHEF_ARTIFACT_BUILD"
            Write-Host "=========================================="

            Write-Host "Verifying AWS connectivity..."
            $identity = aws sts get-caller-identity --output json | ConvertFrom-Json
            if ($LASTEXITCODE -ne 0) {
              Write-Host "ERROR: AWS connectivity failed"
              exit 1
            }
            Write-Host "AWS Account: $($identity.Account)"
            Write-Host "AWS User   : $($identity.Arn)"

            Write-Host "Verifying Terraform state bucket..."
            aws s3 ls s3://$env:STATE_BUCKET --region $env:AWS_REGION | Out-Null
            if ($LASTEXITCODE -ne 0) {
              Write-Host "ERROR: State bucket $env:STATE_BUCKET not accessible"
              exit 1
            }
            Write-Host "State bucket OK: $env:STATE_BUCKET"
            Write-Host "Pre-flight checks passed"
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
              Write-Host "Fetching latest Terraform artifact from Nexus..."
              $listUrl = "$env:NEXUS_URL/service/rest/v1/components?repository=$env:TF_REPO"

              curl.exe -s -u "$env:NEXUS_USER`:$env:NEXUS_PASS" -o tf_components.json $listUrl
              if ($LASTEXITCODE -ne 0) { Write-Host "ERROR: Failed to query Nexus"; exit 1 }

              $response = Get-Content tf_components.json | ConvertFrom-Json
              if ($response.items.Count -eq 0) { Write-Host "ERROR: No artifacts found in $env:TF_REPO"; exit 1 }

              $latest = $response.items |
                        Sort-Object { $_.assets[0].lastModified } -Descending |
                        Select-Object -First 1

              $tfFile = $latest.assets[0].path.Split("/")[-1]
              Remove-Item tf_components.json -Force -ErrorAction SilentlyContinue
            } else {
              $tfFile = "terraform-infra-$env:TF_ARTIFACT_BUILD.tar.gz"
            }

            Write-Host "Downloading: $tfFile"
            curl.exe -s -w "`nHTTP Status: %{http_code}`n" `
              -u "$env:NEXUS_USER`:$env:NEXUS_PASS" `
              -o "artifacts/$tfFile" `
              "$env:NEXUS_URL/repository/$env:TF_REPO/$tfFile"

            if ($LASTEXITCODE -ne 0) { Write-Host "ERROR: Download failed"; exit 1 }

            Write-Host "Extracting $tfFile..."
            tar -xzf "artifacts/$tfFile" -C terraform/
            if ($LASTEXITCODE -ne 0) { Write-Host "ERROR: Extraction failed"; exit 1 }

            if (!(Test-Path "terraform/main.tf")) {
              Write-Host "ERROR: Terraform artifact corrupted - main.tf not found"
              exit 1
            }

            Write-Host "Terraform artifact ready"
            Get-ChildItem terraform/ | Select-Object Name
          '''
        }
      }
    }

    stage('Download Chef Artifact') {
      when {
        expression { return params.RUN_CHEF && !params.DESTROY }
      }
      steps {
        withCredentials([
          usernamePassword(
            credentialsId: 'nexus-credentials',
            usernameVariable: 'NEXUS_USER',
            passwordVariable: 'NEXUS_PASS'
          )
        ]) {
          powershell '''
            New-Item chef -ItemType Directory -Force | Out-Null

            if ($env:CHEF_ARTIFACT_BUILD -eq "latest") {
              Write-Host "Fetching latest Chef artifact from Nexus..."
              $listUrl = "$env:NEXUS_URL/service/rest/v1/components?repository=$env:CHEF_REPO"

              curl.exe -s -u "$env:NEXUS_USER`:$env:NEXUS_PASS" -o chef_components.json $listUrl
              if ($LASTEXITCODE -ne 0) { Write-Host "ERROR: Failed to query Nexus"; exit 1 }

              $response = Get-Content chef_components.json | ConvertFrom-Json
              if ($response.items.Count -eq 0) { Write-Host "ERROR: No artifacts found in $env:CHEF_REPO"; exit 1 }

              $latest = $response.items |
                        Sort-Object { $_.assets[0].lastModified } -Descending |
                        Select-Object -First 1

              $chefFile = $latest.assets[0].path.Split("/")[-1]
              Remove-Item chef_components.json -Force -ErrorAction SilentlyContinue
            } else {
              $chefFile = "chef-config-$env:CHEF_ARTIFACT_BUILD.tar.gz"
            }

            Write-Host "Downloading: $chefFile"
            curl.exe -s -w "`nHTTP Status: %{http_code}`n" `
              -u "$env:NEXUS_USER`:$env:NEXUS_PASS" `
              -o "artifacts/$chefFile" `
              "$env:NEXUS_URL/repository/$env:CHEF_REPO/$chefFile"

            if ($LASTEXITCODE -ne 0) { Write-Host "ERROR: Download failed"; exit 1 }

            Write-Host "Extracting $chefFile..."
            tar -xzf "artifacts/$chefFile" -C chef/
            if ($LASTEXITCODE -ne 0) { Write-Host "ERROR: Extraction failed"; exit 1 }

            Write-Host "Chef artifact ready"
            Get-ChildItem chef/ | Select-Object Name
          '''
        }
      }
    }

    stage('Validate Module Dependencies') {
      when {
        expression { return !params.DESTROY }
      }
      steps {
        powershell '''
          # Apply SELECT_ALL_MODULES override first
          if ($env:SELECT_ALL_MODULES -eq "true") {
            Write-Host "SELECT_ALL_MODULES enabled - overriding all modules to true"
            $env:DEPLOY_SECURITY     = "true"
            $env:DEPLOY_NETWORKING   = "true"
            $env:DEPLOY_COMPUTE      = "true"
            $env:DEPLOY_LOADBALANCER = "true"
            $env:DEPLOY_DATABASE     = "true"
          }

          Write-Host "=========================================="
          Write-Host "  MODULE SELECTION SUMMARY"
          Write-Host "=========================================="
          Write-Host "Select All   : $env:SELECT_ALL_MODULES"
          Write-Host "Security     : $env:DEPLOY_SECURITY"
          Write-Host "Networking   : $env:DEPLOY_NETWORKING"
          Write-Host "Compute      : $env:DEPLOY_COMPUTE"
          Write-Host "Load Balancer: $env:DEPLOY_LOADBALANCER"
          Write-Host "Database     : $env:DEPLOY_DATABASE"
          Write-Host "=========================================="

          $errors = @()

          # Compute requires Security
          if ($env:DEPLOY_COMPUTE -eq "true" -and $env:DEPLOY_SECURITY -ne "true") {
            $errors += "Compute requires Security to be enabled"
          }

          # LoadBalancer requires Security + Compute
          if ($env:DEPLOY_LOADBALANCER -eq "true" -and $env:DEPLOY_SECURITY -ne "true") {
            $errors += "LoadBalancer requires Security to be enabled"
          }
          if ($env:DEPLOY_LOADBALANCER -eq "true" -and $env:DEPLOY_COMPUTE -ne "true") {
            $errors += "LoadBalancer requires Compute to be enabled"
          }

          # Database requires Security
          if ($env:DEPLOY_DATABASE -eq "true" -and $env:DEPLOY_SECURITY -ne "true") {
            $errors += "Database requires Security to be enabled"
          }

          # At least one module must be selected
          $anySelected = (
            $env:DEPLOY_SECURITY     -eq "true" -or
            $env:DEPLOY_NETWORKING   -eq "true" -or
            $env:DEPLOY_COMPUTE      -eq "true" -or
            $env:DEPLOY_LOADBALANCER -eq "true" -or
            $env:DEPLOY_DATABASE     -eq "true"
          )
          if (-not $anySelected) {
            $errors += "At least one module must be selected"
          }

          if ($errors.Count -gt 0) {
            Write-Host "=========================================="
            Write-Host "  MODULE DEPENDENCY ERRORS"
            Write-Host "=========================================="
            foreach ($err in $errors) {
              Write-Host "ERROR: $err"
            }
            Write-Host "=========================================="
            exit 1
          }

          Write-Host "Module dependency validation passed"
        '''
      }
    }

    stage('Generate main.tf') {
      when {
        expression { return !params.DESTROY }
      }
      steps {
        powershell '''
          Set-Location terraform

          # Apply SELECT_ALL_MODULES override
          if ($env:SELECT_ALL_MODULES -eq "true") {
            $env:DEPLOY_SECURITY     = "true"
            $env:DEPLOY_NETWORKING   = "true"
            $env:DEPLOY_COMPUTE      = "true"
            $env:DEPLOY_LOADBALANCER = "true"
            $env:DEPLOY_DATABASE     = "true"
          }

          Write-Host "Generating main.tf for environment: $env:ENVIRONMENT"

          $mainTf = @"
# Auto-generated by Jenkins deploy pipeline
# Environment  : $env:ENVIRONMENT
# Generated    : $(Get-Date -Format "yyyy-MM-dd HH:mm:ss")
# Build        : $env:BUILD_NUMBER
# Select All   : $env:SELECT_ALL_MODULES
# Modules      : Security=$env:DEPLOY_SECURITY Networking=$env:DEPLOY_NETWORKING Compute=$env:DEPLOY_COMPUTE ALB=$env:DEPLOY_LOADBALANCER DB=$env:DEPLOY_DATABASE

"@

          if ($env:DEPLOY_SECURITY -eq "true") {
            $mainTf += @"
module "security" {
  source      = "./modules/security"
  vpc_id      = var.vpc_id
  app_name    = var.app_name
  environment = var.environment
}

"@
            Write-Host "Added: security module"
          }

          if ($env:DEPLOY_NETWORKING -eq "true") {
            $mainTf += @"
module "networking" {
  source      = "./modules/networking"
  vpc_id      = var.vpc_id
  app_name    = var.app_name
  environment = var.environment
}

"@
            Write-Host "Added: networking module"
          }

          if ($env:DEPLOY_COMPUTE -eq "true") {
            $mainTf += @"
module "compute" {
  source        = "./modules/compute"
  ami_id        = var.ami_id
  instance_type = var.instance_type
  subnet_id     = var.ec2_subnet_id
  app_name      = var.app_name
  environment   = var.environment
  key_pair_name = var.key_pair_name
  ec2_sg_id     = module.security.ec2_sg_id
}

"@
            Write-Host "Added: compute module"
          }

          if ($env:DEPLOY_LOADBALANCER -eq "true") {
            $mainTf += @"
module "loadbalancer" {
  source      = "./modules/loadbalancer"
  vpc_id      = var.vpc_id
  subnet_ids  = var.subnet_ids
  app_name    = var.app_name
  environment = var.environment
  alb_sg_id   = module.security.alb_sg_id
  instance_id = module.compute.instance_id
}

"@
            Write-Host "Added: loadbalancer module"
          }

          if ($env:DEPLOY_DATABASE -eq "true") {
            $mainTf += @"
module "database" {
  source                = "./modules/database"
  app_name              = var.app_name
  environment           = var.environment
  subnet_ids            = var.subnet_ids
  db_sg_id              = module.security.db_sg_id
  multi_az              = var.multi_az
  backup_retention_days = var.backup_retention_days
}

"@
            Write-Host "Added: database module"
          }

          # Write generated main.tf without BOM
          $utf8NoBom = New-Object System.Text.UTF8Encoding $false
          [System.IO.File]::WriteAllText("$PWD/main.tf", $mainTf, $utf8NoBom)

          Write-Host ""
          Write-Host "=========================================="
          Write-Host "  GENERATED main.tf"
          Write-Host "=========================================="
          Get-Content main.tf
          Write-Host "=========================================="
        '''
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
              $env:PATH = "$env:TF_PATH;$env:PATH"
              Set-Location terraform

              Write-Host "Running terraform init..."
              Write-Host "State key: mywebapp/$env:ENVIRONMENT/terraform.tfstate"

              terraform init `
                -backend-config="bucket=$env:STATE_BUCKET" `
                -backend-config="key=mywebapp/$env:ENVIRONMENT/terraform.tfstate" `
                -backend-config="region=$env:AWS_REGION" `
                -backend-config="dynamodb_table=terraform-state-lock" `
                -backend-config="encrypt=true" `
                -upgrade

              if ($LASTEXITCODE -ne 0) {
                Write-Host "ERROR: terraform init failed"
                exit 1
              }
              Write-Host "Terraform init complete"
            '''
          }
        }
      }
    }

    stage('Check Existing State') {
      steps {
        withCredentials([[
          $class: 'AmazonWebServicesCredentialsBinding',
          credentialsId: 'aws-credentials'
        ]]) {
          powershell '''
            $env:PATH = "$env:TF_PATH;$env:PATH"
            Set-Location terraform

            Write-Host "Checking existing state for: $env:ENVIRONMENT"
            $stateList = terraform state list 2>$null

            if ($stateList) {
              Write-Host "Existing resources in state:"
              $stateList
              Write-Host "---"
              Write-Host "Total: $($stateList.Count) resources tracked"
            } else {
              Write-Host "No existing state - fresh deployment"
            }
          '''
        }
      }
    }

    stage('Terraform Validate') {
      steps {
        powershell '''
          $env:PATH = "$env:TF_PATH;$env:PATH"
          Set-Location terraform

          Write-Host "Running terraform fmt check..."
          terraform fmt -check
          if ($LASTEXITCODE -ne 0) {
            Write-Host "ERROR: Run terraform fmt to fix formatting"
            exit 1
          }

          Write-Host "Running terraform validate..."
          terraform validate
          if ($LASTEXITCODE -ne 0) {
            Write-Host "ERROR: terraform validate failed"
            exit 1
          }

          Write-Host "Terraform validation passed"
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
            $env:PATH = "$env:TF_PATH;$env:PATH"
            Set-Location terraform

            # Verify tfvars file exists
            $tfvarsFile = "tfvars/$env:ENVIRONMENT.tfvars"
            if (!(Test-Path $tfvarsFile)) {
              Write-Host "ERROR: tfvars file not found: $tfvarsFile"
              Write-Host "Available tfvars:"
              Get-ChildItem tfvars/ -ErrorAction SilentlyContinue | Select-Object Name
              exit 1
            }

            Write-Host "Using tfvars: $tfvarsFile"

            if ($env:DESTROY -eq "true") {
              Write-Host "Planning DESTROY for $env:ENVIRONMENT..."
              $planArgs = @("plan", "-destroy", "-var-file=$tfvarsFile", "-out=tfplan")
            } else {
              Write-Host "Planning APPLY for $env:ENVIRONMENT..."
              $planArgs = @("plan", "-var-file=$tfvarsFile", "-out=tfplan")
            }

            & terraform $planArgs
            if ($LASTEXITCODE -ne 0) {
              Write-Host "ERROR: terraform plan failed"
              exit 1
            }

            Write-Host "Terraform plan complete"
          '''
        }
        archiveArtifacts artifacts: 'terraform/tfplan'
      }
    }

    stage('Approval Gate') {
      when {
        expression {
          return params.ENVIRONMENT == 'prod' ||
                 params.ENVIRONMENT == 'qa'   ||
                 params.DESTROY == true
        }
      }
      steps {
        script {
          def msg = params.DESTROY ?
            "CONFIRM DESTROY of ${params.ENVIRONMENT} infrastructure?" :
            "Approve deployment to ${params.ENVIRONMENT.toUpperCase()}?"
          input message: msg,
                ok: params.DESTROY ? "Yes Destroy" : "Approve",
                submitter: "admin"
        }
      }
    }

    stage('Terraform Apply') {
      when {
        expression { return params.APPLY_TERRAFORM }
      }
      steps {
        retry(2) {
          withCredentials([[
            $class: 'AmazonWebServicesCredentialsBinding',
            credentialsId: 'aws-credentials'
          ]]) {
            powershell '''
              $env:PATH = "$env:TF_PATH;$env:PATH"
              Set-Location terraform

              if ($env:DESTROY -eq "true") {
                Write-Host "DESTROYING infrastructure for $env:ENVIRONMENT..."
              } else {
                Write-Host "APPLYING infrastructure for $env:ENVIRONMENT..."
              }

              terraform apply -auto-approve tfplan
              if ($LASTEXITCODE -ne 0) {
                Write-Host "ERROR: terraform apply failed"
                exit 1
              }

              if ($env:DESTROY -ne "true") {
                terraform output -json | Out-File -FilePath "$env:WORKSPACE\tf_outputs.json" -Encoding UTF8
                Write-Host "Outputs saved to tf_outputs.json"
                Write-Host "State saved: s3://$env:STATE_BUCKET/mywebapp/$env:ENVIRONMENT/terraform.tfstate"
                Get-Content "$env:WORKSPACE\tf_outputs.json"
              } else {
                Write-Host "Infrastructure destroyed"
                Write-Host "State updated: s3://$env:STATE_BUCKET/mywebapp/$env:ENVIRONMENT/terraform.tfstate"
              }
            '''
          }
        }
      }
    }

    stage('Wait for EC2 Bootstrap') {
      when {
        expression { return params.APPLY_TERRAFORM && params.DEPLOY_COMPUTE && !params.DESTROY }
      }
      steps {
        withCredentials([[
          $class: 'AmazonWebServicesCredentialsBinding',
          credentialsId: 'aws-credentials'
        ]]) {
          powershell '''
            $outputs = Get-Content "$env:WORKSPACE\tf_outputs.json" | ConvertFrom-Json
            $ec2Id   = $outputs.ec2_instance_id.value

            Write-Host "Waiting for EC2 $ec2Id to pass status checks..."

            aws ec2 wait instance-status-ok `
              --instance-ids $ec2Id `
              --region $env:AWS_REGION

            if ($LASTEXITCODE -ne 0) {
              Write-Host "ERROR: EC2 did not pass status checks"
              exit 1
            }

            Write-Host "EC2 healthy - waiting 90s for bootstrap to complete..."
            Start-Sleep -Seconds 90
            Write-Host "Ready for Chef configuration"
          '''
        }
      }
    }

    stage('Upload Chef Cookbooks') {
      when {
        expression { return params.RUN_CHEF && params.DEPLOY_COMPUTE && !params.DESTROY }
      }
      steps {
        withCredentials([[
          $class: 'AmazonWebServicesCredentialsBinding',
          credentialsId: 'aws-credentials'
        ]]) {
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
        expression { return params.RUN_CHEF && params.DEPLOY_COMPUTE && !params.DESTROY }
      }
      steps {
        withCredentials([[
          $class: 'AmazonWebServicesCredentialsBinding',
          credentialsId: 'aws-credentials'
        ]]) {
          powershell '''
            $outputs = Get-Content "$env:WORKSPACE\tf_outputs.json" | ConvertFrom-Json
            $ec2Id   = $outputs.ec2_instance_id.value
            Write-Host "Running Chef on EC2: $ec2Id"

            $commandId = aws ssm send-command `
              --document-name "AWS-RunPowerShellScript" `
              --instance-ids $ec2Id `
              --parameters commands=["
                aws s3 cp s3://$env:S3_BUCKET/chef/cookbooks C:/chef/cookbooks --recursive --region $env:AWS_REGION;
                if (`$LASTEXITCODE -ne 0) { exit 1 };
                chef-client --local-mode --runlist 'recipe[webserver]' --chef-license accept;
                if (`$LASTEXITCODE -ne 0) { exit 1 }
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
    expression { return params.APPLY_TERRAFORM && params.DEPLOY_LOADBALANCER && !params.DESTROY }
  }
  steps {
    powershell '''
      if (!(Test-Path "$env:WORKSPACE\tf_outputs.json")) {
        Write-Host "No tf_outputs.json found - skipping verification"
        exit 0
      }

      $outputs = Get-Content "$env:WORKSPACE\tf_outputs.json" | ConvertFrom-Json

      if (-not $outputs.alb_dns_name -or -not $outputs.alb_dns_name.value) {
        Write-Host "ALB not deployed - skipping health check"
        exit 0
      }

      $albDns = $outputs.alb_dns_name.value
      $ec2Ip  = $outputs.ec2_public_ip.value

      Write-Host "=========================================="
      Write-Host "  DEPLOYMENT SUMMARY"
      Write-Host "=========================================="
      Write-Host "Environment: $env:ENVIRONMENT"
      Write-Host "ALB DNS    : http://$albDns"
      Write-Host "EC2 IP     : $ec2Ip"
      Write-Host "Health     : http://$albDns/health"
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
        Write-Host "Check EC2 manually at http://$ec2Ip"
      }
    '''
  }
}

  }

  post {
    success {
      script {
        def action = params.DESTROY ? ":boom: *INFRASTRUCTURE DESTROYED*" : ":rocket: *DEPLOYMENT SUCCESS*"
        def modules = []
        if (params.SELECT_ALL_MODULES)  modules << "All Modules"
        else {
          if (params.DEPLOY_SECURITY)     modules << "Security"
          if (params.DEPLOY_NETWORKING)   modules << "Networking"
          if (params.DEPLOY_COMPUTE)      modules << "Compute"
          if (params.DEPLOY_LOADBALANCER) modules << "LoadBalancer"
          if (params.DEPLOY_DATABASE)     modules << "Database"
        }
        slackSend(
          channel: '#jenkins',
          color: params.DESTROY ? 'warning' : 'good',
          message: """
${action}
*Job:* ${env.JOB_NAME}
*Build:* #${env.BUILD_NUMBER}
*Environment:* ${params.ENVIRONMENT}
*Modules:* ${modules.join(', ')}
*Duration:* ${currentBuild.durationString}
*State:* s3://terraform-state-364829013514/mywebapp/${params.ENVIRONMENT}/terraform.tfstate
*URL:* ${env.BUILD_URL}
          """.trim()
        )
      }
    }
    failure {
      slackSend(
        channel: '#jenkins',
        color: 'danger',
        message: """
:fire: *DEPLOYMENT FAILED*
*Job:* ${env.JOB_NAME}
*Build:* #${env.BUILD_NUMBER}
*Environment:* ${params.ENVIRONMENT}
*Duration:* ${currentBuild.durationString}
*State preserved:* s3://terraform-state-364829013514/mywebapp/${params.ENVIRONMENT}/terraform.tfstate
*URL:* ${env.BUILD_URL}console
        """.trim()
      )
    }
    unstable {
      slackSend(
        channel: '#jenkins',
        color: 'warning',
        message: """
:warning: *BUILD UNSTABLE*
*Job:* ${env.JOB_NAME}
*Build:* #${env.BUILD_NUMBER}
*Environment:* ${params.ENVIRONMENT}
*Duration:* ${currentBuild.durationString}
*URL:* ${env.BUILD_URL}
        """.trim()
      )
    }
    always {
      cleanWs()
    }
  }

}