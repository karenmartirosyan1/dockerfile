 # Description
 ## Problem
 Instead of manually executing a ``docker build`` command in the Dind Server Pod we need to have a dynamically provisioned pod which will connect to the Dind Server Pod and execute ``docker build`` command.
 ## Solution
 This solution introduces a Dind Client Pod defined the Jenkinsfile as a Jenkins agent which connects to the Dind Server and executes a command. 
 # Components
 * ##  Dind Server
    For this case Dind Server must listen on port ``2375``. Here are K8S Dind Server Pod, PV, PVC and Service examples defined in ``yaml``.
 ``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: dind-test
  labels:  
    app: dind-test
spec:  
  containers:
  - name: dind-test
    image: docker:18.05-dind
    volumeMounts: 
    - name: dind-storage
      mountPath: /var/lib/docker
    securityContext:
      privileged: true
  volumes:
  - name: dind-storage
    persistentVolumeClaim:
      claimName: myclaim
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv
spec:
  capacity:
    storage: 2Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Recycle
  hostPath:
    path: /tmp/volume
--- 
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myclaim
spec:
  accessModes:
    - ReadWriteMany
  volumeMode: Filesystem
  resources:
    requests:
      storage: 2Gi
  volumeName: pv 
---
apiVersion: v1
kind: Service
metadata:
  name: dind-svc
  namespace: test
spec:
  selector:
    app: dind-test
  ports:
    - protocol: TCP
      port: 2375
      targetPort: 2375
 ``` 
* ## Jenkinsfile
    Here is the Jenkinsfile example with Dind Client Pod as an agent, which will connect to the Dind Server. Docker Client executes a ``docker build`` command in the Dind Server.

    **Note**: ``defaultContainer`` is set to `dind`, so if you want to run a command inside a ``jnlp`` container you must specify it.
    ```groovy
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
        stage('Testing some command in jnlp container') {
          steps { 
              container('jnlp') {
                  sh 'git --version'
                }                               
            }                  
        }
        stage("Tesing Dind pod") {
          steps {
              script {
                  docker.build(" test_image", "-f Dockerfile .")
                    }
                }
            }
        }
    }
