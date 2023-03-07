# RoboMaker Demo ROS1

## 创建开发环境

创建EC2并通过Cloud9连接EC2，作为RoboMaker的开发环境。参见: https://aws.amazon.com/blogs/robotics/robotics-development-in-aws-cloud9/

#### 1. 创建CloudFormation Stack

* Template：ec2_instance_builder.json
* 创建IAM Instance Profile `ec2-instance-builder`， SSM将使用其作为EC2 Instance Profile
* 创建Security Group 允许SSH和TCP:8080， SSM将使用其作为EC2 Security Group

#### 2. 通过SSM Document创建EC2

* 创建SSM Document：`ROS1-Cloud9-Instance-Builder`，在Editor内粘贴模板：ROS1-Cloud9-Instance-Builder.yaml
* 执行Document 自动化, 参数如下
    * instanceType： m5.xlarge 
    * volumeSize: 50
    * instanceProfileARN: `arn:aws:iam::<account-id>:instance-profile/ec2-instance-builder` (CloudFormation Stack 输出中找到InstanceProfileARN）
    * cloud9PublicKey：`ssh-rsa XXXXX`（粘贴Cloud9 SSH Key：前往Cloud9服务（https://us-east-1.console.aws.amazon.com/cloud9control/home?region=us-east-1#/create/），Environment type选择Existing Compute，点击`Copy `k`ey to clipboard`按钮）
    * securityGroupIds：sg-xxxxx (CloudFormation Stack 输出中找到SecurityGroupId）
* 执行完成后，在Output中找到EC2 Instance Id

#### 3. 连接Cloud9与EC2

* 前往EC2 Instance页面，搜索Document 创建的Instance Id，复制PublicIPv4DNS
* 前往Cloud9服务，创建新环境，Environment type选择Existing Compute，其他参数如下
    * User: `ubuntu`
    * Host: `[ec2-xx-xx-xx-xx.compute-1.amazonaws.com](http://ec2-50-17-100-26.compute-1.amazonaws.com/)` (EC2 Instance PublicIPv4DNS)
    * Additional details > Path to Node.js binary: `/home/ubuntu/.nvm/versions/node/v12.22.11/bin/node`
* 创建环境，完成后打开，即可连接EC2 Instance

## 仿真任务

### 1. 获得样例代码

* Dependencies

```
git clone https://github.com/aws-robotics/aws-robomaker-sample-application-helloworld.git helloworld && cd helloworld
vcs import robot_ws < robot_ws/.rosinstall 
vcs import simulation_ws < simulation_ws/.rosinstall
```

* Dockerfile / entrypoint.sh 已经创建好

### 2. 构建robot / simulation applications 容器镜像

```
export robotapp=robomaker-helloworld-robot-app 
export simapp=robomaker-helloworld-sim-app
```

```
DOCKER_BUILDKIT=1 docker build . \
--build-arg ROS_DISTRO=melodic \
--build-arg LOCAL_WS_DIR=./robot_ws \
--build-arg APP_NAME=helloworld-robot-app \
-t $robotapp
```

```
DOCKER_BUILDKIT=1 docker build . \
--build-arg GAZEBO_VERSION=gazebo-9 \
--build-arg ROS_DISTRO=melodic \
--build-arg LOCAL_WS_DIR=./simulation_ws \
--build-arg APP_NAME=helloworld-sim-app \
-t $simapp`
```

```
# if permission denied error about /var/run/docker.sock
sudo usermod -aG docker $USER
sudo chmod 666 /var/run/docker.sock
```

```
docker images
```

### 3. 将镜像发布到ECR

```
export region=us-east-1
aws configure set region $region

export account=$(aws sts get-caller-identity --query "Account" --output text)
export ecruri=$account.dkr.ecr.$region.amazonaws.com
```

```
# Create ECR repositories
aws ecr get-login-password --region $region | docker login --username AWS --password-stdin $ecruri 
aws ecr create-repository --repository-name $robotapp 
aws ecr create-repository --repository-name $simapp
```

```
# Tag Docker Images with ECR URI
docker tag $robotapp $ecruri/$robotapp:latest 
docker tag $simapp $ecruri/$simapp:latest
```

```
# Upload to ECR
docker push $ecruri/$robotapp 
docker push $ecruri/$simapp
```

```
# List uploaded images
docker push $ecruri/$robotapp 
docker push $ecruri/$simapp
```

### 4. 创建机器人和方针应用

```
aws robomaker create-robot-application \
--name $robotapp \
--robot-software-suite name=General \
--environment uri=$ecruri/$robotapp:latest
```

```
aws robomaker create-simulation-application \
--name $simapp \
--simulation-software-suite name=SimulationRuntime \
--robot-software-suite name=General \
--environment uri=$ecruri/$simapp:latest
```

```
robo_iam_role=arn:aws:iam::$account:role/robomaker-sample-role
robot_app_arn=$(aws robomaker list-robot-applications --filters name=name,values=$robotapp --query "robotApplicationSummaries"[0]."arn" --output text)
sim_app_arn=$(aws robomaker list-simulation-applications --filters name=name,values=$simapp --query "simulationApplicationSummaries"[0]."arn" --output text)


cat > create_simulation_job.json << EOF
{
    "maxJobDurationInSeconds": 3600,
    "iamRole": "$robo_iam_role",
    "robotApplications": [
        {
            "application": "$robot_app_arn",
            "applicationVersion": "\$LATEST",
            "launchConfig": {
                "environmentVariables": {
                    "ROS_IP": "ROBOMAKER_ROBOT_APP_IP",
                    "ROS_MASTER_URI": "http://ROBOMAKER_ROBOT_APP_IP:11311",
                    "GAZEBO_MASTER_URI": "http://ROBOMAKER_SIM_APP_IP:11345"
                },
                "streamUI": false,
                "command": [
                    "roslaunch", "hello_world_robot", "rotate.launch"
                ]
            },
            "tools": [
                {
                    "streamUI": true,
                    "name": "robot-terminal",
                    "command": "/entrypoint.sh && xfce4-terminal",
                    "streamOutputToCloudWatch": true,
                    "exitBehavior": "RESTART"
                }
            ]
        }
    ],
    "simulationApplications": [
        {
            "application": "$sim_app_arn",
            "launchConfig": {
                "environmentVariables": {
                  "ROS_IP": "ROBOMAKER_SIM_APP_IP",
                  "ROS_MASTER_URI": "http://ROBOMAKER_ROBOT_APP_IP:11311",
                  "GAZEBO_MASTER_URI": "http://ROBOMAKER_SIM_APP_IP:11345",
                  "TURTLEBOT3_MODEL":"waffle_pi"
                },
                "streamUI": true,
                "command": [
                    "roslaunch", "aws-robomaker-small-house-world", "small_house.launch"
                ]
            },
            "tools": [
                {
                    "streamUI": true,
                    "name": "gzclient",
                    "command": "/entrypoint.sh && gzclient",
                    "streamOutputToCloudWatch": true,
                    "exitBehavior": "RESTART"
                }
            ]
        }
    ]
}
EOF

```

```
aws robomaker create-simulation-job --cli-input-json file://create_simulation_job.json
```



