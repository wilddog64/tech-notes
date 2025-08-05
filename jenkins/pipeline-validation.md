# Requirements for Pipeline Validation

What	 | Why it matters
------ | ------
Jenkins ≥ 2.54|That’s when the modern CLI client was introduced.
Pipeline Model Definition plugin ≥ 1.1.7|This plugin registers the declarative-linter CLI command.
CLI access (SSH or jenkins-cli.jar)|The command is available over either transport as long as the user has Overall/Read plus Job/Configure (or Job/Read) permissions.

## Example command

### Using SSH
```bash

    # Discover the random SSH port (if one was auto-assigned)
    curl -sI https://jenkins.example.com/login | grep -i x-ssh-endpoint

    # Suppose it printed 53801 …
    ssh -p 53801 jenkins@example.com declarative-linter < Jenkinsfile
```
### Using jenkins-cli.jar

```bash
java -jar jenkins-cli.jar \
     -s https://jenkins.example.com/ \
     -auth user:APITOKEN \
     declarative-linter < Jenkinsfile

```
add `-webSocket` if you want to use the WebSocket transport instead of HTTP.

### Raw HTTP endpoint

use curl like this:

```bash
curl -X POST -u user:APITOKEN \
  -F "jenkinsfile=< Jenkinsfile" \
  https://jenkins.example.com/pipeline-model-converter/validate
