# Pipeline Migration

## Linting
A playbook that commonly used that Jenkins upgrade hasn't broken anything -
especially the declarative pipeline that depend on dozens of plugins.

### Start with a safe controller

Step|Why
---|---
Spin up a clone of production (container volume + $JENKINS\_HOME) on a side port or a disposable VM.|Lets you test plugin/class-data migrations without endangering the real instance.
Pin the exact core & plugin set you intend to ship.|Mixing versions in test vs prod defeats the purpose.
Disable outbound webhooks / credentials in the clone (e.g., use a dummy secrets file).|Prevents staging jobs from pushing artefacts or hitting prod infra.

### Smoke-test the controller

1. Check the first boot log for red stack traces. For checking contaner, `docker logs <container-id>`.
2. From the log, check to see if each plugin installed and loaded successfully.
3. Validate every Jenkisfile up front

   ``` bash
   # list all jobs in the controller
   java -jar jenkins-cli.jar -s $URL -auth $U:$T list-jobs | \
   while read job; do
     java -jar jenkins-cli.jar -s $URL -auth $U:$T declarative-linter \
          --job "$job" || echo "LINT FAILED: $job"
   done
   ```
4. Execerise plugin code paths
   Replay/re-run representative jobs. From container Jenkins CLI:

   ``` groovy
   // Groovy script from Script Console
   Jenkins.instance.getAllItems(org.jenkinsci.plugins.workflow.job.WorkflowJob).each {
       println "Triggering smoke build for ${it.fullName}"
       it.scheduleBuild2(0)
   }
   ```
   Watch Stage View or Blue Oean for any red stage means plugin or API incompatibility
