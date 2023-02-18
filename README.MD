```shell
aws-profile-list() {
  cat ~/.aws/config | grep '^\[profile ' | grep -v '^#' | sed -e 's/^\[profile //;s/]$//'
}

aws-login() {
  if [ -z "${1}" ]; then
    echo "Usage: aws-login AWS_PROFILE" 1>&2
    return
  fi

  SSO_START_URL=$(aws configure get "profile.${1}.sso_start_url")

  if [ -z "${SSO_START_URL}" ]; then
    echo "profile (${1}) not configured for sso; Missing sso_start_url" 1>&2
    return
  fi

  ACCESS_TOKEN=$(cat ~/.aws/sso/cache/*.json | jq -r --arg url "${SSO_START_URL}" 'select(.startUrl==$url).accessToken')
  SSO_ACCOUNT_ID=$(aws configure get "profile.${1}.sso_account_id")
  SSO_ROLE_NAME=$(aws configure get "profile.${1}.sso_role_name")
  CREDS=$(aws sso get-role-credentials --account-id "${SSO_ACCOUNT_ID}" --role-name "${SSO_ROLE_NAME}" --access-token "${ACCESS_TOKEN}" --profile "${1}" 2>/dev/null)

  if [ "$?" != "0" ]; then
    aws sso login --profile "${1}" &>/dev/null
    ACCESS_TOKEN=$(cat ~/.aws/sso/cache/*.json | jq -r --arg url "${SSO_START_URL}" 'select(.startUrl==$url).accessToken')
    CREDS=$(aws sso get-role-credentials --account-id "${SSO_ACCOUNT_ID}" --role-name "${SSO_ROLE_NAME}" --access-token "${ACCESS_TOKEN}" --profile "${1}")
  fi

  export AWS_ACCESS_KEY_ID=$(jq -r '.roleCredentials.accessKeyId' <<< "${CREDS}")
  export AWS_SECRET_ACCESS_KEY=$(jq -r '.roleCredentials.secretAccessKey' <<< "${CREDS}")
  export AWS_SESSION_TOKEN=$(jq -r '.roleCredentials.sessionToken' <<< "${CREDS}")
  export AWS_PROFILE="${1}"
}

function ec2 {
  local filter
  local EC2ID

  case "${1}" in
    10*)
    filter="Name=network-interface.addresses.private-ip-address,Values=${1}"
    ;;
    i-*)
    filter="Name=instance-id,Values=${1}"
    ;;
    *)
    filter="Name=tag-value,Values=${1}"
    ;;
  esac

  EC2ID=$(aws ec2 describe-instances --filters "${filter}" Name=instance-state-name,Values=running --query 'Reservations[].Instances[].InstanceId' --output text)

  [ -n "${EC2ID}" ] && aws ssm start-session \
  --target "${EC2ID}" \
  --profile "${AWS_PROFILE}" \
  --document-name AWS-StartInteractiveCommand \
  --parameters 'command=["sudo -i -u ssm-user bash"]' || echo "Instance not found"
}

function ssh-ssm {
  local filter
  local EC2ID

  case "${1}" in
    10*)
    filter="Name=network-interface.addresses.private-ip-address,Values=${1}"
    ;;
    i-*)
    filter="Name=instance-id,Values=${1}"
    ;;
    *)
    filter="Name=tag-value,Values=${1}"
    ;;
  esac

  EC2ID=$(aws ec2 describe-instances --filters "${filter}" Name=instance-state-name,Values=running --query 'Reservations[].Instances[].InstanceId' --output text)

  [ -n "${EC2ID}" ] && ssh ec2-user@"${EC2ID}" || echo "Instance not found"
}

function proxyrdp {
  local filter
  local EC2ID

  case "${1}" in
    10*)
    filter="Name=network-interface.addresses.private-ip-address,Values=${1}"
    ;;
    i-*)
    filter="Name=instance-id,Values=${1}"
    ;;
    *)
    filter="Name=tag-value,Values=${1}"
    ;;
  esac

  EC2ID=$(aws ec2 describe-instances --filters "${filter}" Name=instance-state-name,Values=running --query 'Reservations[].Instances[].InstanceId' --output text)

  [ -n "${EC2ID}" ] && aws ssm start-session \
  --target "${EC2ID}" \
  --profile "${AWS_PROFILE}" \
  --document-name AWS-StartPortForwardingSession \
  --parameters "localPortNumber=3389,portNumber=3389" || echo "Instance not found"
}

function proxyport {
  local filter
  local EC2ID

  case "${1}" in
    10*)
    filter="Name=network-interface.addresses.private-ip-address,Values=${1}"
    ;;
    i-*)
    filter="Name=instance-id,Values=${1}"
    ;;
    *)
    filter="Name=tag-value,Values=${1}"
    ;;
  esac

  EC2ID=$(aws ec2 describe-instances --filters "${filter}" Name=instance-state-name,Values=running --query 'Reservations[].Instances[].InstanceId' --output text)

  [ -n "${EC2ID}" ] && aws ssm start-session \
  --target "${EC2ID}" \
  --profile "${AWS_PROFILE}" \
  --document-name AWS-StartPortForwardingSession \
  --parameters "localPortNumber=${2},portNumber=${3}" || echo "Instance not found"
}

function proxyansiblewin {
  local filter
  local EC2ID

  case "${1}" in
    10*)
    filter="Name=network-interface.addresses.private-ip-address,Values=${1}"
    ;;
    i-*)
    filter="Name=instance-id,Values=${1}"
    ;;
    *)
    filter="Name=tag-value,Values=${1}"
    ;;
  esac

  EC2ID=$(aws ec2 describe-instances --filters "${filter}" Name=instance-state-name,Values=running --query 'Reservations[].Instances[].InstanceId' --output text)

  [ -n "${EC2ID}" ] && aws ssm start-session \
  --target "${EC2ID}" \
  --profile "${AWS_PROFILE}" \
  --document-name AWS-StartPortForwardingSession \
  --parameters "localPortNumber=5986,portNumber=5986" || echo "Instance not found"
}

function dumpec2 {
  local filter

  case "${1}" in
    10*)
    filter="Name=network-interface.addresses.private-ip-address,Values=${1}"
    ;;
    i-*)
    filter="Name=instance-id,Values=${1}"
    ;;
    *)
    filter="Name=tag-value,Values=${1}"
    ;;
  esac

  aws ec2 describe-instances --filters "${filter}" --query 'Reservations[].Instances[]'
}

function win_reset_password {
  local filter
  local EC2ID

  case "${1}" in
    10*)
    filter="Name=network-interface.addresses.private-ip-address,Values=${1}"
    ;;
    i-*)
    filter="Name=instance-id,Values=${1}"
    ;;
    *)
    filter="Name=tag-value,Values=${1}"
    ;;
  esac

  EC2ID=$(aws ec2 describe-instances --filters "${filter}" Name=instance-state-name,Values=running --query 'Reservations[].Instances[].InstanceId' --output text)

  aws ssm send-command \
  --document-name "AWSSupport-RunEC2RescueForWindowsTool" \
  --document-version "14" \
  --targets "[{\"Key\":\"InstanceIds\",\"Values\":[\"${EC2ID}\"]}]" \
  --parameters '{"Command":["ResetAccess"],"Parameters":["alias/aws/ssm"]}' \
  --timeout-seconds 600 \
  --max-concurrency "50" \
  --max-errors "0" \
  --region us-east-1 | jq '.Command.CommandId'
}

function ssm_results {
  local filter
  local EC2ID

  case "${1}" in
    10*)
    filter="Name=network-interface.addresses.private-ip-address,Values=${1}"
    ;;
    i-*)
    filter="Name=instance-id,Values=${1}"
    ;;
    *)
    filter="Name=tag-value,Values=${1}"
    ;;
  esac

  [ -z "${2}" ] && echo "missing command id - $0 <instance> <command id>" && return

  EC2ID=$(aws ec2 describe-instances --filters "${filter}" Name=instance-state-name,Values=running --query 'Reservations[].Instances[].InstanceId' --output text)

  aws ssm get-command-invocation ---instance-id "${EC2ID}" --command-id "${2}" --region us-east-1
}

function ecr_login {
  local AWS_ACCOUNT_ID

  AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query 'Account' --output text)
  aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin "${AWS_ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com"
}

function rds_proxy_port {
  local filter
  local EC2ID

  case "${1}" in
    10*)
    filter="Name=network-interface.addresses.private-ip-address,Values=${1}"
    ;;
    i-*)
    filter="Name=instance-id,Values=${1}"
    ;;
    *)
    filter="Name=tag-value,Values=${1}"
    ;;
  esac

  EC2ID=$(aws ec2 describe-instances --filters "${filter}" Name=instance-state-name,Values=running --query 'Reservations[].Instances[].InstanceId' --output text)

  RDSARN=$(aws resourcegroupstaggingapi get-resources --resource-type-filters rds:db --tag-filters "Key=Name,Values=${2}" --query 'ResourceTagMappingList[].ResourceARN' --output text
)

  RDSID=$(aws rds describe-db-instances --filters "Name=db-instance-id,Values=${RDSARN}" | jq -r '.DBInstances[].Endpoint.Address')

  [ -n "${EC2ID}" ] && aws ssm start-session \
    --region us-east-1 \
    --target "${EC2ID}" \
    --document-name AWS-StartPortForwardingSessionToRemoteHost \
    --parameters host="${RDSID}",portNumber="${4}",localPortNumber="${3}" || echo "Instance not found"
}
```