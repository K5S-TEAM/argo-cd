pipeline {
    agent any

    environment {
        GIT_ID = "LittleSamakFox"
        GIT_URL_JENKINS = "github.com/LittleSamakFox/cicd-test-springboot.git"
        GIT_URL_ARGO = "github.com/LittleSamakFox/argo-cd-test.git"
        GIT_CREDENTIALS_ID = "GIT-Credentials"
        AWS_REGION = "ap-northeast-2"
        AWS_CREDENTIALS_ID = "AWS-Credentials"
        ECR_BASE_URL = "262213740647.dkr.ecr.ap-northeast-2.amazonaws.com"
        ECR_IMAGE_URL = "262213740647.dkr.ecr.ap-northeast-2.amazonaws.com/k5s_ecr"
        IMAGE_TAG = "latest"
    }

    stages {
        stage('Git Clone') {
            steps {
                script {
                    try {
                        git url: "https://$GIT_URL_JENKINS",
                            branch: "main"
                            credentialsId: "$GIT_CREDENTIALS_ID"
                        env.cloneResult=true
                    } catch (error) {
                        print(error)
                        env.cloneResult=false
                        currentBuild.result = 'FAILURE' 
                    }
                }
            }
            post {
                success {
                    echo "Repository clone success !"
                }
                failure{
                    echo "Repository clone failure !"
                }
            }
        }
        stage('Build JAR by Gradle'){
            when {
                expression {
                    return env.cloneResult ==~ /(?i)(Y|YES|T|TRUE|ON|RUN)/
                }                
            }
            steps {
                script {
                    try {
												sh "sudo cp /home/ec2-user/k5s/auth/application.yaml ./src/main/resources/application.yaml"
                        sh "chmod +x gradlew"
                        sh "./gradlew clean build -x test"
                        sh "ls -al ./build"
                        env.gradleBuildResult=true
                    } catch (error) {
                        print(error)
                        echo "Remove Deploy Files"
                        sh './gradlew clean'
                        env.gradleBuildResult=false
                        currentBuild.result = 'FAILURE'
                    }
                }
            }
            post {
                success {
                    echo "Gradle jar build success !"
                }
                failure{
                    echo "Gradle jar build failure !"
                }
            }
        }
        stage('Build Image'){
            when {
                expression {
                    return env.gradleBuildResult ==~ /(?i)(Y|YES|T|TRUE|ON|RUN)/
                }                
            }
            steps {
                script {
                    try {
                        sh """
                        #!/bin/bash
                        cat>Dockerfile<<-EOF
FROM openjdk:11-jre-slim
COPY ./build/libs/*-SNAPSHOT.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar", "--spring.config.name=", "/home/ec2-user/property/test.yml"]
EOF
                        """
                        k5s_image = docker.build("$ECR_IMAGE_URL")
                        env.imageBuildResult=true
                    } catch (error) {
                        print(error)
                        echo "Remove Deploy Files"
                        sh './gradlew clean'
                        env.imageBuildResult=false
                        currentBuild.result = 'FAILURE'
                    }
                }
            }
            post {
                success {
                    echo "image success !"
                }
                failure{
                    echo "image failure !"
                }
            }
        }
        stage('Push Image to ECR'){
            when {
                expression {
                    return env.imageBuildResult ==~ /(?i)(Y|YES|T|TRUE|ON|RUN)/
                }                
            }
            steps {
                script {
                    try {
                       docker.withRegistry("https://$ECR_BASE_URL", "ecr:$AWS_REGION:$AWS_CREDENTIALS_ID") {
                            k5s_image.push("v${currentBuild.number}")
                            k5s_image.push("latest")
                        }
                        env.ecrPushResult=true
                    } catch (error) {
                        print(error)
                        echo "Remove Deploy Files"
                        sh './gradlew clean'
                        env.ecrPushResult=false
                        currentBuild.result = 'FAILURE'
                    }
                }
            }
            post {
                success {
                    echo "ECR Push success !"
                }
                failure{
                    echo "ECR Push failure !"
                }
            }
        }
        stage('build manifest'){
            when {
                expression {
                    return env.ecrPushResult ==~ /(?i)(Y|YES|T|TRUE|ON|RUN)/
                }                
            }
            steps {
                script {
                    try {
                        git url: "https://$GIT_URL_ARGO",
                            branch: "main"
                            credentialsId: "$GIT_CREDENTIALS_ID"
                        sh "rm -rf /var/lib/jenkins/workspace/${currentBuild.projectName}/*"
                        sh """
                        #!/bin/bash
                        cat>deployment.yaml<<-EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: k5smovie-demo-deploy
  labels:
    app: k5smovie-demo-deploy
spec:
  replicas: 2
  selector:
    matchLabels:
      app: k5smovie-demo
  template:
    metadata:
      labels:
        app: k5smovie-demo
    spec:
      containers:
      - name: demo-spring-boot
        image: $ECR_IMAGE_URL:v${currentBuild.number}
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8080
EOF
                        cat>service.yaml<<-EOF
apiVersion: v1
kind: Service
metadata:
  name: k5smovie-demo-svc
  labels:
    app: k5smovie-demo-svc
spec:
  type: NodePort
  selector:
    app: k5smovie-demo
	ports:
    - name: http
      protocol : TCP
      port : 80
      targetPort : 8080
    - name: https
      protocol : TCP
      port : 443
      targetPort : 8443
EOF
                        cat>ingress.yaml<<-EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: k5smovie-demo-ingress # 인그레스 이름 정하기
  namespace: default    # 설치할 네임스페이스
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/group.name: k5s-ingress
    alb.ingress.kubernetes.io/group.order: '2'
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/certificate-arn: [ROUTE 53 인증서]
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}, {"HTTP":80}]'
    alb.ingress.kubernetes.io/ssl-redirect: '443'
spec:
  rules:
  - http:
      paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: k5smovie-demo-svc
              port:
                number: 80  # (이부분도 고정)
EOF"""
                        sh """
                        git add deployment.yaml
                        git add service.yaml
                        git add ingress.yaml
                        git commit -m "update deployment image_version ${currentBuild.number}"
                        git push https://$GIT_ID:$GIT_TOCKEN@$GIT_URL_ARGO
                        """
                        sh "rm -rf /var/lib/jenkins/workspace/${currentBuild.projectName}/*"
                        env.buildManifestResult=true
                    } catch (error) {
                        print(error)
                        echo "Remove Deploy Files"
                        sh './gradlew clean'
                        env.buildManifestResult=false
                        currentBuild.result = 'FAILURE'
                    }
                }
            }
            post {
                success {
                    echo "build manifest success !"
                }
                failure{
                    echo "build manifest failure !"
                }
            }
        }
    }
}
