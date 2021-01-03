pipeline {
  // 어떤 노드를 사용할 것인가?
  agent any

  // 지정된 주기로 파이프라인 실행
  triggers {
    pollSCM('*/3 * * * * ')
  }

  // 환경 변수 설정
  environment {
    AWS_ACCESS_KEY_ID = credentials('awsAccessKeyId')
    AWS_SECRET_ACCESS_KEY = credentials('awsSecretAccessKey')
    AWS_DEFAULT_REGION = 'ap-northeast-2'
    HOME = '.'
  }

  stages {
    // -- 레포지토리 다운 --
    stage('Prepare') {
      agent any

      // # git 에서 레포지토리 다운 받기
      steps {
        echo "Lets start Long Journey ! ENV: ${ENV}"
        echo 'Clonning Repository'

        git url: 'https://github.com/carrot-farm/jenkins-test.git',
          branch: 'master',
          credentialsId: 'gittest'
      }

      // # 클론 후 
      post {
        // 성공 시
        success: {
          echo 'Successfully Cloned Repository'
        }

        // 성공, 실패 상관 없음
        alaways {
          echo 'i tried...'
        }

        // 작업 완료 후
        cleanup {
          echo "after all other post condition"
        }
      }
    }

    // -- 프로덕션에서만 실행될 것들 --
    stage('Only for production') {
      when {
        // branch 가 'production'이면서 environment의 APP_ENV = prod인 경우 실행
        branch: 'produciton' 
        environment name: 'APP_ENV', value: 'prod'
        anyOf {
          environment name: 'DEPLOY_TO', value: 'production'
          environment name: 'DEPLOY_TO', value: 'staging'
        }
      }
    }

    // -- aws s3에 파일 업로드 --
    stage('Deploy Frontend') {
      steps {
        echo 'Deploying Frontend'
        // # 프론트엔드 디렉토리의 정적파일들을 S3에 업로드. 이 전에 반드시 EC2 instacne profile을 등록해야함
        // 여기서 린트 등의 다양한 작업 을 함.
        dir('./website') {
          sh '''
          aws s3 sync ./ s3://static-server-bucket/jenkins_test/
          '''
        }
      }

      post {
        // # 성공
        success {
          echo 'Successfully Cloned Repository'

          mail to: 'chohoki@gmail.com',
            subject: "Deploy Frontend Success",
            body: "Successfully deployed frontend!"
        }

        // # 실패 시 
        failure {
          echo 'I failed :<',

          mail to: 'chohoki@gmail.com',
            subject: "Failed Pipeline",
            body: "Something is wrong with deploy frontend"
        }
      }
    }

    // -- 코드 린트 --
    stage('Lint Backend') {
      // # nodejs 도커 이미지 실행
      // Docker plugin 과 Docker Pipeline 플러그인 2개 필요.
      // 실제 프로덕션에서는 글로벌 이미지가 아니라 ECR등에 저장된 이미지 사용.
      agent {
        docker {
          image 'node:latest'
        }
      }

      steps {
        dir('./server') {
          sh '''
          npm install &&
          npm run lint
          '''
        }
      }
    }

    // -- 테스트 --
    stage('Test Backend') {
      // # nodejs 도커 이미지 실행
      agent {
        docker {
          image 'node:latest'
        }
      }
      
      // # 테스트
      steps {
        echo 'Test Backend'

        dir('./server') {
          sh '''
          npm install
          npm run test
          '''
        }
      }
    }

    // -- build --
    stage('Build Backend') {
      agent any

      steps {
        echo 'Build Backend'

        dir('./server') {
          sh """
          docker build . -t server --build-arg env=${PROD}
          """
        }
      }

      post {
        failure {
          // `error` 출력시 다음 state로 넘어가지 않음.
          error 'This pipeline stops here...'
        }
      }
    }

    // -- deploy --
    // ECS나 쿠버네티스 등을 실행
    stage('Deploy Backend') {
      agent any

      steps {
        echo 'Build Backend'

        dir ('./server') {
          // 실행되는 도커 전부 끄기.
          // sh '''
          // docker rm -f $(docker ps -aq)
          // docker run -p 80:80 -d server
          // '''
          // 도커 실행
          sh '''
          docker run -p 80:80 -d server
          '''
        }
      }

      post {
        success {
          mail to: 'chohoki@gmail.com',
            subject: "Deploy Success",
            body: "Successfully deployed"
        }
      }
    }
  }
}