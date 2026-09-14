```sh
command -v aws
aws --version
# macOS / Linux / WSL2
curl -fsSL https://awscli.amazonaws.com/v2/install.sh | bash\nexport PATH="$HOME/.local/bin:$PATH"
# Windows (Git Bash): 공식 MSI 설치 관리자 내려받아 설치
curl -o AWSCLIV2.msi https://awscli.amazonaws.com/AWSCLIV2.msi
MSYS_NO_PATHCONV=1 msiexec.exe /i AWSCLIV2.msi /qn
aws --version
```

```sh
aws configure sso --profile studentXX

# SSO session name: infra-training
# SSO start URL: https://infra-lab-ai.awsapps.com/start
# SSO region: ap-northeast-2
# SSO registration scopes: sso:account:access
# CLI default client Region: ap-northeast-2
# CLI default output format: json
# CLI profile name: studentXX

aws sts get-caller-identity --profile studentXX
```

