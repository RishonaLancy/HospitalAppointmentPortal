pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
        timestamps()
    }

    environment {
        PATH = "C:\\Users\\risho\\AppData\\Local\\Programs\\DockerDesktop\\resources\\bin;${env.PATH}"
        IMAGE = 'rishonalancy/hospital-appointment-portal'
        DEPLOYMENT = 'hospital-appointment-portal'
        SERVICE = 'hospital-appointment-service'
        DEPLOYMENT_ATTEMPTED = 'false'
    }

    stages {
        stage('1. Clone Code') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/RishonaLancy/HospitalAppointmentPortal.git'
            }
        }

        stage('2. Build Docker Image') {
            steps {
                bat 'docker build -t %IMAGE%:%BUILD_NUMBER% .'
            }
        }

        stage('3. Test Application') {
            steps {
                bat '''
                    @echo off
                    set TEST_CONTAINER=hospital-test-%BUILD_NUMBER%

                    docker run -d --name %TEST_CONTAINER% -p 18080:80 %IMAGE%:%BUILD_NUMBER%
                    if errorlevel 1 exit /b 1

                    powershell -NoProfile -ExecutionPolicy Bypass -Command "$ok=$false; for($i=0; $i -lt 10; $i++){ try { $r=Invoke-WebRequest -Uri 'http://localhost:18080' -UseBasicParsing -TimeoutSec 3; if($r.StatusCode -eq 200){$ok=$true; break} } catch {}; Start-Sleep -Seconds 2 }; if(-not $ok){exit 1}"

                    set TEST_RESULT=%ERRORLEVEL%
                    docker rm -f %TEST_CONTAINER% >nul 2>&1
                    exit /b %TEST_RESULT%
                '''
            }
        }

        stage('4. Push Image to Docker Hub') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-credentials',
                        usernameVariable: 'DOCKERHUB_USER',
                        passwordVariable: 'DOCKERHUB_PASSWORD'
                    )
                ]) {
                    bat '''
                        @echo off
                        echo %DOCKERHUB_PASSWORD%| docker login -u "%DOCKERHUB_USER%" --password-stdin
                        if errorlevel 1 exit /b 1
                        docker push %IMAGE%:%BUILD_NUMBER%
                        if errorlevel 1 exit /b 1
                        docker logout
                    '''
                }
            }
        }

        stage('5. Deploy to Kubernetes') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'kubernetes-config',
                        variable: 'KUBECONFIG'
                    )
                ]) {
                    script {
                        bat '''
                            @echo off
                            kubectl apply -f k8s\\service.yaml
                            if errorlevel 1 exit /b 1

                            kubectl get deployment %DEPLOYMENT% >nul 2>&1
                            if errorlevel 1 (
                                kubectl apply -f k8s\\deployment.yaml
                                if errorlevel 1 exit /b 1
                            )
                        '''

                        bat '''
                            @echo off
                            kubectl set image deployment/%DEPLOYMENT% portal=%IMAGE%:%BUILD_NUMBER%
                            if errorlevel 1 exit /b 1
                        '''

                        env.DEPLOYMENT_ATTEMPTED = 'true'

                        bat '''
                            @echo off
                            kubectl rollout status deployment/%DEPLOYMENT% --timeout=180s
                            if errorlevel 1 exit /b 1
                        '''
                    }
                }
            }
        }

        stage('6. Validate Deployed Application') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'kubernetes-config',
                        variable: 'KUBECONFIG'
                    )
                ]) {
                    bat '''
                        @echo off
                        kubectl exec deployment/%DEPLOYMENT% -- wget -q -O - http://127.0.0.1/ > deployed-page.html
                        if errorlevel 1 exit /b 1

                        powershell -NoProfile -ExecutionPolicy Bypass -Command "$p=Get-Content 'deployed-page.html' -Raw; if($p -notmatch 'Dr\\. Meera Shah' -or $p -notmatch 'Hospital Announcements'){Write-Error 'Application content validation failed'; exit 1}"
                    '''
                }
            }
        }

        stage('7. Verify Three Replicas') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'kubernetes-config',
                        variable: 'KUBECONFIG'
                    )
                ]) {
                    bat '''
                        @echo off
                        kubectl get deployment %DEPLOYMENT%
                        kubectl get pods -l app=hospital-appointment-portal
                        kubectl get service %SERVICE%
                    '''
                }
            }
        }
    }

    post {
        failure {
            script {
                if (env.DEPLOYMENT_ATTEMPTED == 'true') {
                    echo 'Pipeline failed after deployment was attempted. Restoring the previous Kubernetes revision.'

                    withCredentials([
                        file(
                            credentialsId: 'kubernetes-config',
                            variable: 'KUBECONFIG'
                        )
                    ]) {
                        bat '''
                            @echo off
                            kubectl rollout undo deployment/%DEPLOYMENT%
                            if errorlevel 1 exit /b 1

                            kubectl rollout status deployment/%DEPLOYMENT% --timeout=180s
                            if errorlevel 1 exit /b 1

                            kubectl get deployment %DEPLOYMENT%
                            kubectl get pods -l app=hospital-appointment-portal
                        '''
                    }
                } else {
                    echo 'Failure occurred before the Kubernetes image update. No rollout undo was attempted.'
                }
            }
        }
    }
}
