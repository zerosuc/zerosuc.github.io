---
title: "Jenkins Pipeline"
date: 2024-08-23T14:51:10-08:00
draft: false
categories: ["Jenkins", "Pipeline"]
tags: ["Jenkins", "Pipeline"]
---

## Jenkins Pipeline

要实现在 Jenkins 中的构建工作，可以有多种方式，我们这里采用比较常用的 Pipeline 这种方式。Pipeline，简单来说，就是一套运行在 Jenkins 上的工作流框架，将原来独立运行于单个或者多个节点的任务连接起来，实现单个任务难以完成的复杂流程编排和可视化的工作。

Jenkins Pipeline 有几个核心概念：

- Node：节点，一个 Node 就是一个 Jenkins 节点，Master 或者 Agent，是执行 Step 的具体运行环境，比如我们之前动态运行的 Jenkins Slave 就是一个 Node 节点
- Stage：阶段，一个 Pipeline 可以划分为若干个 Stage，每个 Stage 代表一组操作，比如：Build、Test、Deploy，Stage 是一个逻辑分组的概念，可以跨多个 Node
- Step：步骤，Step 是最基本的操作单元，可以是打印一句话，也可以是构建一个 Docker 镜像，由各类 Jenkins 插件提供，比如命令：sh 'make'，就相当于我们平时 shell 终端中执行 make 命令一样。

那么我们如何创建 Jenkins Pipline 呢？

- Pipeline 脚本是由 Groovy 语言实现的，但是我们没必要单独去学习 Groovy，当然你会的话最好
- Pipeline 支持两种语法：Declarative(声明式)和 Scripted Pipeline(脚本式)语法
- Pipeline 也有两种创建方法：可以直接在 Jenkins 的 Web UI 界面中输入脚本；也可以通过创建一个 Jenkinsfile 脚本文件放入项目源码库中
- 一般我们都推荐在 Jenkins 中直接从源代码控制(SCMD)中直接载入 Jenkinsfile Pipeline 这种方法

这里我们使用 `podTemplate` 来定义不同阶段使用的的容器；这里给一个 demo

```yam
def dockerreg='10.x.x.x'
podTemplate(nodeSelector: 'nvidia.com/gpu.present=true',cloud: 'kubernetes',label: "vbank-restful", imagePullSecrets: [ 'harbor' ], containers: [
        containerTemplate(name: 'docker', image: "${dockerreg}/common/docker:20.10.18", ttyEnabled: true, alwaysPullImage: true, command: 'cat'),
        containerTemplate(name: 'maven', image: "${dockerreg}/common/maven:3.3.9_8u152", ttyEnabled: true, command: 'cat'),
        containerTemplate(name: 'jnlp', image: "${dockerreg}/devopstools/jenkins/jnlp-slave:4.13.2-1-jdk11", ttyEnabled: true,  args: '${computer.jnlpmac} ${computer.name}'),
        containerTemplate(name: 'sonarcli', image: "${dockerreg}/devopstools/sonarsource/sonar-scanner-cli:4.8.0", ttyEnabled: true, command: 'cat'),
		//python包用于自动化测试时使用
        //containerTemplate(name: 'python', image:"${dockerreg}/tester/python:2.7.13", ttyEnabled: true, alwaysPullImage: false, command: 'cat'),
    ],
    volumes: [
                hostPathVolume(hostPath: '/var/run/docker.sock', mountPath: '/var/run/docker.sock'),
                persistentVolumeClaim(claimName: 'maven-claim', mountPath: '/root/.m2', readOnly: false),
                persistentVolumeClaim(claimName: 'jarfile-claim', mountPath: '/webapps', readOnly: false),
            ],
    )
    {

        node('vbank-restful') {
            try {
                stage('check out viid code release')
				giturl="git@gitlab.xxx.com:yyy/viid/viid.git"
                if (tag_version != "master") {
                    git credentialsId: 'gitvimicro', url: "${giturl}"
                    env.check_to_tag="$tag_version"
                    sh '[ -n "${check_to_tag}" ] &&  git checkout  ${check_to_tag} ||  { echo -e "切换至指定的tag的版本，tag：${check_to_tag} 不存在或为空，请检查输入的tag!" && exit 111; }'
                    git_branch = "$tag_version"
                    tag_name = "$tag_version"

                } else if (branch != "") {
                    git credentialsId: 'gitvimicro', url: "${giturl}", branch: branch
                    git_branch = "$branch"
                    tag_name = ""
                } else {
                     git credentialsId: 'gitvimicro', url: "${giturl}"

                     tag_name = ""
                     git_branch =  sh (script: "git symbolic-ref --short -q HEAD", returnStdout: true).trim()
                }

                stage('get branch name and version digist')
                git_commit = sh (script: "git rev-parse --short HEAD", returnStdout: true).trim()
                last_author = sh (returnStdout: true, script: "git show -s --pretty=%an").trim()
                nowtime = sh (returnStdout: true, script: 'date +%Y%m%d%H%M%S').trim()

                container('sonarcli') {
                    stage('sonarqube check ')
                    withSonarQubeEnv(credentialsId: 'sonarqube') {
						sh "sonar-scanner -Dsonar.projectName=${JOB_NAME} -Dsonar.projectKey=${JOB_NAME} -Dsonar.java.binaries=. -Dsonar.java.source=1.8"
					}
				    timeout(time: 10) {
				        sleep 10
						waitForQualityGate abortPipeline: true
					}
               }
                container("maven") {
                    stage('mvn build')

                    echo "now build revision: ${git_branch} ${git_commit}"
                    sh "mvn clean install -U --settings /usr/share/maven/ref/settings-docker.xml -Dmaven.test.skip=true -Dmaven_addr=${maven_addr}"
					//image search模块单独打包
				    // sh "cd viid-image-search && mvn clean package -U --settings /usr/share/maven/ref/settings-docker.xml -Dmaven.test.skip=true -Dmaven_addr=${maven_addr}"
                    //sh "mvn clean install -U --settings /usr/share/maven/ref/settings-docker.xml -Dmaven_addr=${maven_addr} -DskipTests -am -pl viid-image-search"
                    //sh "cd viid-image-search && mvn clean install -U --settings /usr/share/maven/ref/settings-docker.xml -Dmaven.test.skip=true -Dmaven_addr=${maven_addr}"
                    // viid-log-analyse模块打包
                    // sh "cd viid-log-analyse && mvn clean package -U --settings /usr/share/maven/ref/settings-docker.xml -Dmaven.test.skip=true -Dmaven_addr=${maven_addr}"
                    //sh "mvn clean install -U --settings /usr/share/maven/ref/settings-docker.xml -Dmaven_addr=${maven_addr} -DskipTests -am -pl viid-log-analyse"


                    stage('backup build targets')
                    if (tag_name ==~ /release-(.*)/) {
                        version = nowtime + '-' + tag_name
                    } else {
                        version = nowtime + "-" + git_commit
                    }
                    /*
					//viid-notify
					sh 'if [ ! -d /webapps/viid/viid-notify ]; then mkdir -p /webapps/viid/viid-notify; fi'
                    sh 'cp viid-notify/target/*.jar /webapps/viid/viid-notify/viid-notify-' + version + '.jar'
                    sh 'cp viid-notify/src/main/resources/application.properties /webapps/viid/viid-notify/application-' + version + '.properties'

					//redis-db
					sh 'if [ ! -d /webapps/viid/redis-db ]; then mkdir -p /webapps/viid/redis-db; fi'
					sh 'cp viid-data-redis-sync-to-db/target/*.jar /webapps/viid/redis-db/redis-db-' + version + '.jar'
					sh 'cp viid-data-redis-sync-to-db/src/main/resources/application.properties /webapps/viid/redis-db/application-' + version + '.properties'


                    //viid-sample
                    sh """
					if ls viid-sampling/target/viid-sampling-*.jar > /dev/null 2>&1;
					then
						if [ ! -d /webapps/viid/viid-sampling ]; then mkdir -p /webapps/viid/viid-sampling; fi
					    cp viid-sampling/target/viid-sampling-*.jar /webapps/viid/viid-sampling/viid-sampling-${version}.jar \
					    && cp viid-sampling/src/main/resources/application.properties /webapps/viid/viid-sampling/application-${version}.properties;
					fi
					"""

					//viid-searchimage
                    sh """
					if ls viid-image-search/target/viid-image-search-*.jar > /dev/null 2>&1;
					then
						if [ ! -d /webapps/viid/viid-image-search ]; then mkdir -p /webapps/viid/viid-image-search; fi
					    cp viid-image-search/target/viid-image-search-*.jar /webapps/viid/viid-image-search/viid-image-search-${version}.jar \
					    && cp viid-image-search/src/main/resources/application.yml /webapps/viid/viid-image-search/application-${version}.yml;
					fi
					"""
					*/
                }



                container("docker") {
					stage 'docker build'
					if (tag_name ==~ /release-(.*)/) {
						tag_name = (tag_name =~ /release-(.*)/)[0][1]
                        //platform = "linux/amd64,linux/arm64"
                    } else {
                        tag_name = git_commit
                        //platform = "linux/amd64"
                    }



                    sh """
                    export registry=10.200.82.51:80
                    export tag_name=${tag_name}
                    export os_version=${os_version}
                    wget http://10.200.82.35/packages/kubeconfig/config -O /config
                    export KUBECONFIG=/config
                    KUBECONFIG=/config docker buildx create --bootstrap --name builder --driver=kubernetes '--driver-opt="image=moby/buildkit:buildx-stable-1","namespace=jenkins"' --use
                    docker login -u jenkins -p Jenkins123 \${registry}
                    docker buildx bake -f docker-compose.yml --set *.platform=${platform}  --set *.args.REGISTRY=${registry} --set *.args.GITHASH=${git_commit} --set *.args.VERSION=${version} --set *.args.OS_VERSION=${os_version} --push
                    """



				}

            } catch(e) {
                currentBuild.result = "FAILED"
                throw e
            } finally {
                // def svnrevision = sh (script: "svn info --show-item last-changed-revision", returnStdout: true).trim()
                stage("sendemail")
                notifyBuild(currentBuild.result, git_branch, version)
            }
        }

    }

def notifyBuild(String buildStatus = 'STARTED', String gitbranch = 'unknown', String svnrevision='unkonwn') {
    buildStatus = buildStatus ?: 'SUCCESS'
	gitbranch = gitbranch ?: 'unkonwn'
    svnrevision = svnrevision ?: 'unknown'
    // def colorName = 'RED'
    // def colorCode = '#FF0000'
    if (tag_name ==~ /release-(.*)/) {
        subject ="${JOB_NAME} - git # ${tag_name} - ${buildStatus}"
    } else {
        subject = "${JOB_NAME} - git # ${gitbranch} ${svnrevision} ${last_author}- ${buildStatus}"
    }
    def CAUSE= getLastBuildCause()

    if (env.gitlabMergeRequestId) {
    details = '''
         (本邮件由程序自动发送，请勿回复！)<br/>
            这是vbank_restful部署程序发布的版本及构建信息<br/><hr/>
            项目名称：${JOB_NAME}<br/><hr/>
            构建编号： ${BUILD_NUMBER}<br/><hr/>
            构建原因： ${CAUSE}<br/><hr/>
            构建分支：${gitbranch}<br/><hr/>
            构建描述：${gitlabMergeRequestTitle}<br/><hr/>
            构建日志地址： <a href="${BUILD_URL}console">${BUILD_URL}console</a><br/>
            构建地址： <a href="$BUILD_URL">${BUILD_URL}</a><br/><hr/>
        '''
    } else {
         details = '''
         (本邮件由程序自动发送，请勿回复！)<br/>
            这是vbank_restful部署程序发布的版本及构建信息<br/><hr/>
            项目名称：${JOB_NAME}<br/><hr/>
            构建编号： ${BUILD_NUMBER}<br/><hr/>
            构建原因： ${CAUSE}<br/><hr/>
            构建日志地址： <a href="${BUILD_URL}console">${BUILD_URL}console</a><br/>
            构建地址： <a href="$BUILD_URL">${BUILD_URL}</a><br/><hr/>
        '''
    }

    if (tag_name ==~ /release-(.*)/) {
        emailext(
          subject: subject,
          mimeType: 'text/html',
          body: details,
          to: emailname
        )
    } else {
        emailext(
          subject: subject,
          mimeType: 'text/html',
          body: details,
          to: emailname,
          //attachLog: true,
          //attachmentsPattern: 'web_page/test_reports/*.html'
          )
    }
}

@NonCPS
def getLastBuildCause() {
    def causes = currentBuild.rawBuild.getCauses()
    return causes.last().getShortDescription()
}


```

问题 1：

提示不能解析我们的 GitLab 域名，这是因为我们的域名都是自定义的，我们可以通过在 CoreDNS 中添加自定义域名解析来解决这个问题（如果你的域名是外网可以正常解析的就不会出现这个问题了）：

```
$ kubectl edit cm coredns -n kube-system
apiVersion: v1
data:
  Corefile: |
    .:53 {
        log
        errors
        health {
          lameduck 5s
        }
        ready
        hosts {  # 添加自定义域名解析
          192.168.0.100 git.k8s.local
          192.168.0.100 jenkins.k8s.local
          192.168.0.100 harbor.k8s.local
          fallthrough
        }
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           upstream
           fallthrough in-addr.arpa ip6.arpa
        }
        prometheus :9153
        forward . /etc/resolv.conf
        cache 30
        loop
        reload
        loadbalance
    }
kind: ConfigMap
......
```

修改完成后，隔一小会儿，CoreDNS 就会自动热加载，我们就可以在集群内访问我们自定义的域名了。然后肯定没有权限，所以需要配置帐号认证信息。
