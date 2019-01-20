#!/usr/bin/env groovy

pipeline {
	agent {
    label 'ubuntu'
  }
	options {
		ansiColor( 'xterm' )
		disableConcurrentBuilds()
		skipStagesAfterUnstable()
		timestamps()
	}
	tools {
    	'org.jenkinsci.plugins.terraform.TerraformInstallation' 'terraform-0.11.10'
		'com.cloudbees.jenkins.plugins.customtools.CustomTool' 'awscli'
  }
	stages {
		stage( 'Create Instance' ) {
			steps {
				withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: '607434ae-08a4-4578-9e41-5d400fff2cf9']]) {
          withEnv(["PATH+TF=${tool 'terraform-0.11.10'}"]) {
          	sh """
							env | sort
							terraform -v
							aws --version
							cd recover-all/terraform/

							echo "***  starting terraform"
							terraform init
							terraform apply -auto-approve

							mkdir -p ../../inventory

							#create ansible host file
							echo "[master1]\n\$(terraform output ip-node-1)\n\n[all-masters]\n\$(terraform output ip-node-1)\n\$(terraform output ip-node-2)\n\$(terraform output ip-node-3)" > ../../inventory/hosts
			        cat ../../inventory/hosts

							#wait untill all nodes are up
							aws ec2 wait instance-status-ok --instance-ids \$(terraform output rancher_1) --region \$(terraform output aws_region)
							aws ec2 wait instance-status-ok --instance-ids \$(terraform output rancher_2) --region \$(terraform output aws_region)
							aws ec2 wait instance-status-ok --instance-ids \$(terraform output rancher_3) --region \$(terraform output aws_region)
							aws ec2 wait instance-status-ok --instance-ids \$(terraform output worker_1) --region \$(terraform output aws_region)
							aws ec2 wait instance-status-ok --instance-ids \$(terraform output worker_2) --region \$(terraform output aws_region)
							aws ec2 wait instance-status-ok --instance-ids \$(terraform output worker_3) --region \$(terraform output aws_region)
							aws ec2 wait instance-status-ok --instance-ids \$(terraform output worker_4) --region \$(terraform output aws_region)
							aws ec2 wait instance-status-ok --instance-ids \$(terraform output worker_5) --region \$(terraform output aws_region)
							aws ec2 wait instance-status-ok --instance-ids \$(terraform output worker_6) --region \$(terraform output aws_region)

							echo "*** Deployment complete"

							#passes and creates credentials to aws script
							echo "#!/bin/bash\ncd /home/tusimple/\n#fixes s3 error which allows bigger files to be sent \nsudo sysctl -p \n\n#configures aws cli\nprintf '${AWS_ACCESS_KEY_ID}\n${AWS_SECRET_ACCESS_KEY}\n\n\n' | aws configure \n" > ../roles/kube_config/files/backups.sh

						"""
						withCredentials([sshUserPrivateKey(credentialsId: '194405d6-e950-4bd9-b185-b47644d012f0', keyFileVariable: 'pemfile')]) {
	    				sh """
								pem=\$(cat '${pemfile}')

								cd recover-all/terraform/
								terraform init

								#create cluster.yml File

								echo "---\nnodes:\n  - address: \$(terraform output ip-node-1)\n    user: tusimple\n    role:\n    - controlplane\n    - etcd\n    ssh_key: |-\n\$pem\n\n  - address: \$(terraform output ip-node-2)\n    user: tusimple\n    role:\n    - controlplane\n    - etcd\n    ssh_key: |-\n\$pem\n\n  - address: \$(terraform output ip-node-3)\n    user: tusimple\n    role:\n    - controlplane\n    - etcd\n    ssh_key: |-\n\$pem\n\n  - address: \$(terraform output worker-1-ip)\n    user: tusimple\n    role:\n    - worker\n    ssh_key: |-\n\$pem\n\n  - address: \$(terraform output worker-2-ip)\n    user: tusimple\n    role:\n    - worker\n    ssh_key: |-\n\$pem\n\n  - address: \$(terraform output worker-3-ip)\n    user: tusimple\n    role:\n    - worker\n    ssh_key: |-\n\$pem\n\n  - address: \$(terraform output worker-4-ip)\n    user: tusimple\n    role:\n    - worker\n    ssh_key: |-\n\$pem\n\n  - address: \$(terraform output worker-5-ip)\n    user: tusimple\n    role:\n    - worker\n    ssh_key: |-\n\$pem\n\n  - address: \$(terraform output worker-6-ip)\n    user: tusimple\n    role:\n    - worker\n    ssh_key: |-\n\$pem\n\nservices:\n  etcd:\n    snapshot: true #enables recurring\n    creation: 1h0s #increments between each snapshot\n    retention: 24h #time snapshot saved before purged\n\n" > ../roles/kube_config/files/cluster.yml

							"""
						}
        	}
        }
			}
		}
		stage( 'prepare masters' ){
			steps {
				ansiblePlaybook(
                      credentialsId: 'ubuntu-agent',
					  					disableHostKeyChecking: true,
                      inventory: 'inventory/hosts',
                      playbook: 'recover-all/prepare.yml',
                      colorized: true
				)
			}
		}
		stage( 'configure' ){
			steps {
				ansiblePlaybook(
											credentialsId: 'ubuntu-agent',
					  					disableHostKeyChecking: true,
                      inventory: 'inventory/hosts',
                      playbook: 'recover-all/kube.yml',
                      colorized: true
				)
			}
		}
	}
}
