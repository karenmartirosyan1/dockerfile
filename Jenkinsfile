pipeline {
  agent {
    kubernetes {
      defaultContainer 'dind'
      yaml """\
        apiVersion: v1
        kind: Pod
        metadata:
          labels:
            jenkins/kube-default: "true"
            app: jenkins
            component: agent
          namespace: jenkins
        spec:
          containers:
            - name: jnlp
              image: jenkinsci/jnlp-slave
              imagePullPolicy: Always
              env:
              - name: POD_IP
                valueFrom:
                  fieldRef:
                    fieldPath: status.podIP
            - name: dind
              image: docker:18.05-dind
              securityContext:
              env:
              - name: DOCKER_HOST
                value: tcp://dind-svc.karen-test.svc.cluster.local:2375              
                privileged: true
              command:
              - sh 
              - -c
              - tail -f /dev/null
        """.stripIndent()                    
    }
  }
    
    stages {
        stage('Run') {
          steps { 
              container('jnlp') {
                  sh 'git --version'
                                }
                script { 
                  sh 'docker pull mysql' 
                    }
                    
                  
        }    
              
        }
        
        
    }
}
