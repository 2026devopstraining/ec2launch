pipeline {
    agent any

    environment {
        AWS_ACCESS_KEY_ID     = credentials('aws-access-key-id')
        AWS_SECRET_ACCESS_KEY = credentials('aws-secret-access-key')
        AWS_DEFAULT_REGION    = 'us-west-2'

        // ---- Customize these ----
        AMI_ID        = 'ami-0320940581663281e'  
        INSTANCE_TYPE = 't2.micro'
        KEY_NAME      = 'EpsilonKey'         // your existing EC2 key pair name
        // SG_NAME       = 'training-ec2-sg'
        INSTANCE_NAME = 'training-ec2-west2'
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/YOUR_USERNAME/ec2-training-awscli.git'
            }
        }

        stage('Create Security Group') {
            steps {
                sh '''
                    # Check if SG already exists to avoid duplicate error
                    SG_ID=$(aws ec2 describe-security-groups \
                        --filters "Name=group-name,Values=${SG_NAME}" \
                        --query "SecurityGroups[0].GroupId" \
                        --output text 2>/dev/null)

                    if [ "$SG_ID" = "None" ] || [ -z "$SG_ID" ]; then
                        echo "Creating Security Group..."
                        SG_ID=$(aws ec2 create-security-group \
                            --group-name ${SG_NAME} \
                            --description "Training EC2 Security Group" \
                            --query "GroupId" \
                            --output text)

                        # Allow SSH inbound
                        aws ec2 authorize-security-group-ingress \
                            --group-id $SG_ID \
                            --protocol tcp \
                            --port 22 \
                            --cidr 0.0.0.0/0

                        echo "Security Group created: $SG_ID"
                    else
                        echo "Security Group already exists: $SG_ID"
                    fi

                    echo $SG_ID > sg_id.txt
                '''
            }
        }

        stage('Launch EC2 Instance') {
            steps {
                sh '''
                    SG_ID=$(cat sg_id.txt)

                    INSTANCE_ID=$(aws ec2 run-instances \
                        --image-id ${AMI_ID} \
                        --instance-type ${INSTANCE_TYPE} \
                        --key-name ${KEY_NAME} \
                        --security-group-ids $SG_ID \
                        --user-data file://userdata.sh \
                        --tag-specifications \
                            "ResourceType=instance,Tags=[{Key=Name,Value=${INSTANCE_NAME}},{Key=Environment,Value=training},{Key=Owner,Value=jenkins}]" \
                        --query "Instances[0].InstanceId" \
                        --output text)

                    echo "Launched Instance ID: $INSTANCE_ID"
                    echo $INSTANCE_ID > instance_id.txt
                '''
            }
        }

        stage('Wait for EC2 Running State') {
            steps {
                sh '''
                    INSTANCE_ID=$(cat instance_id.txt)
                    echo "Waiting for instance $INSTANCE_ID to be in running state..."

                    aws ec2 wait instance-running --instance-ids $INSTANCE_ID

                    echo "Instance is now RUNNING!"
                '''
            }
        }

        stage('Print EC2 Details') {
            steps {
                sh '''
                    INSTANCE_ID=$(cat instance_id.txt)

                    PUBLIC_IP=$(aws ec2 describe-instances \
                        --instance-ids $INSTANCE_ID \
                        --query "Reservations[0].Instances[0].PublicIpAddress" \
                        --output text)

                    STATE=$(aws ec2 describe-instances \
                        --instance-ids $INSTANCE_ID \
                        --query "Reservations[0].Instances[0].State.Name" \
                        --output text)

                    echo "=============================="
                    echo "Instance ID : $INSTANCE_ID"
                    echo "Public IP   : $PUBLIC_IP"
                    echo "State       : $STATE"
                    echo "Region      : us-west-2"
                    echo "=============================="
                '''
            }
        }
    }

    post {
        success {
            echo 'EC2 instance successfully launched in us-west-2!'
        }
        failure {
            echo 'Pipeline failed. Check the logs above.'
        }
    }
}
