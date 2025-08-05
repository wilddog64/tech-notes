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

5. Check for subtler regressions

Check|Command
-----|-------
Agent connections still work|Agents should reconnect automatically; verify `Manage Nodes` show them online
Log recorders stay quite| Look for repetitive `ClassNotFoundException`, `RejectedAccessException`, CPS method mismatches
System tests run green|End-to-end job that compiles, test & achives artefacts
Performance conters|Compare GC pauses, queue length, thread count to pre-upgrade baselines

6. Understand plugin/-core limits

* Update Center JSON only publishes compatibility metadata for Jenkins ≥ 2.500.x
* Controllers older than that often can't resolve the latest plugin versions. So update core first, then plugins
* Remove plugins flagged as deprecated or unmaintained for long-term support

7. When everything is ready
* Tag the tested image as production
* BAckup the old $JENKINS_HOME and plugin directory so you can roll back if needed
* Perform the same upgrade on the live controller during a maintenance window
* Re-run smoke jobs immediately after the upgrade, and rollback if they fail
