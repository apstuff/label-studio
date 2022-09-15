# LabelStudio

## URLs
- prod: http://apb-labelstudio.us-east-1.elasticbeanstalk.com/
- dev: http://apb-labelstudio-dev.us-east-1.elasticbeanstalk.com/

## Quickstart
```bash
git clone git@github.com:apstuff/label-studio.git
cd label-studio
git checkout eb-dev
eb init
# Region: us-east-1
# Application to use: apb-labelstudio
# Default environment, apb-labelstudio-dev
# CodeCommit: No
eb deploy
```
- This links the `eb-dev` branch to the `apb-labelstudio-dev` environment
- Make changes to the code, commit them, and then use `eb deploy` to deploy a new version
- To set up the release branch use these commands
```bash
git checkout eb-release
eb use apb-labelstudio
```

## Initial Setup
### Deployment
- We use AWS Elastic Beanstalk (eb) to deploy labelstudio
- eb can take a docker compose and create an app from it
    - ec2 instances, load balancer, ssh access, etc
- We start from labelstudio's official compose file
    - https://raw.githubusercontent.com/heartexlabs/label-studio/ec3cfec/docker-compose.yml
- Replace the postgres DB in the compose file with an AWS RDS
- Mount an EFS volume that will hold all our data
- We'll have two environments, prod and dev, with separate resources (DB, volumes, etc)
    - apb-labelstudio
    - apb-labelstudio-dev

### Setup
- Install aws CLI and configure credentials
```bash
brew install awscli
aws configure
```

- Install eb CLI
```bash
brew install aws-elasticbeanstalk
```

```bash
aws configure list-profiles
# choose profile from aws configure list-profiles, omit for default
export AWS_DEFAULT_PROFILE=apb
eb init --profile=apb
eb create
```
- Region: us-east-1
- Application name: apb-labelstudio
- Platform: Docker
- Platform branch: Docker
- CodeCommit: N
- SSH: Yes
- Environment name: apb-labelstudio
- DNS prefix: apb-labelstudio
- load balancer type: application
- Spot fleet: Yes (defaults)

#### Setup Database
- Using AWS RDS
- Go to console and follow this guide
  - https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/using-features.managing.db.html#environments-cfg-rds-create
- engine: postgres
- engine version: 13.7
- instance class: db.t3.micro
- storage: 5GB
- user/password: [in 1PW]

#### Health Checks
- Health checks are defined in `.ebextensions/health-check.config`
- Fallback:
    - Labelstudio runs on :8080 responds with a 302 code
    - Modify health monitor to these values
    ```bash
    eb config
    # modify aws:elasticbeanstalk:environment:process:default
    ```

#### File Storage
- Persistent EFS volume to hold imported data
- Create EFS volume
    - defined in `.ebextensions/02-storage-efs-createfilesystem.config`
- Mount EFS Volume
    - defined in `.ebextensions/03-storage-efs-mountfilesystem.config`

#### Deployment
- Only committed changes are deployed
  - Can be overridden with `--staged` flag
```bash
eb deploy
```

#### Set up Labelstudio username
- SSH into one of the environment's instances
    - `eb ssh`
- Create user
    - `docker exec -it current_app_1 label-studio --username <USER> --password <PASSWORD>`
    - Kill process, exit ssh session
    - This will write the credentials to the DB
