# Use Jenkins replay to validate pipeline migration
Replay is the cheapest way to test pipeline after Jenkins upgrade

Why it is cheap|What it does not catch
---|---
Zero authoring effort. Simply re-running scripts that already exist; no need to write unit test| Logic that depends on `external state` will still pass if step is stubbed or skipped by the old snapshot
No SCM checkout. Replay uses the Groovy byte-code captured in original build, so you're not aiting on Git or Maven|Drift between snapshot and HEAD. IF someone refactored the shared library after that build, replay won't exercise the new code path
Runs in parallel on the same controller. A loop through 200 jobs usually finishes in minutes because they're mostly queueiong stage metadata, not compiling code|`Agent-side regressions`. If the new Jenkins swarm agent jar broke, replay will still handshake with old already-connected agents
Reuses the original parameters & environment. Perfect for quickly surfacing `ClassNotFoundd` or symbol-lookup errors caused by new plugin versions|`Enviroment variables` or credentials that have rotated will cause false failures (good to kno, but noisy if you expect them)

Although `Replay` button is meant for interactive use, you can still replay builds in an automated way via Jenkins' groovy API or its (undocumented) REST API. Below are three approaches that commonly use, from simplest to most controlled

# Re-run a pipeline exactly as it was (no edits)
If all you need is to `run the last build again with same Jenkinsfile and parameters,` the easier path is to:

* `Use the rebuil-or-Retry plugin` (rebuild or pipeline-retry-aborter). They expose `.../rebuild` and `.../retry` endpoints you can hit with curl, e.g.:
   ```bash
   # Rebuild last completed run of job foo
   CRUMB=$(curl -s -u $USER:$TOKEN "$JENKINS_URL/crumbIssuer/api/json" | jq -r '.crumbRequestField+":"+.crumb')
   curl -X POST -H "$CRUMB" -u $USER:$TOKEN \
        "$JENKINS_URL/job/foo/lastCompletedBuild/rebuild"

   ```
   This method does not use replay at all; therefor, no sandbox or CPS migration overhead

* Trigger Replay remotely (same Groovy script, new build)
  Replay records the pipeline script that ran in buil N and schedules N + 1 using that snapshot.
  You can drive that from the outside by POSTing to the hidden form endpoint:
   ```bash
   build=123              # build you want to replay
   job="foo"
   script="$(curl -s -u $USER:$TOKEN "$JENKINS_URL/job/$job/$build/replay/pipelineScript")"

   CRUMB=$(curl -s -u $USER:$TOKEN "$JENKINS_URL/crumbIssuer/api/xml?xpath=concat(//crumbRequestField,\":\",//crumb)")

   curl -s -u $USER:$TOKEN -H "$CRUMB" -X POST \
        --data-urlencode "script=$script" \
        --data "Submit=Run" \
        "$JENKINS_URL/job/$job/$build/replay/submit"

   ```
   _Notes_
   1. Only if users have `Job/Replay` permission this can work
   2. The build N + 1 will appear with a _Replay Cause_ in the UI, exactly as if you clicked the button
   3. To iterate over the last build of every pipeline job, wrap the two `curl` in a shell loop

* Drive Replay from inside the controller (Groovy script console)
  If you have admin rights, the you can do this entirely inside Jenkins,
  ```groovy
  import jenkins.model.*
  import org.jenkinsci.plugins.workflow.cps.replay.ReplayAction

  Jenkins.instance.allItems(
      org.jenkinsci.plugins.workflow.job.WorkflowJob
  ).each { job ->
      def run = job.getLastCompletedBuild()
      if (run != null) {
          def replay = run.getAction(ReplayAction)
          if (replay != null) {
              println "Replaying ${job.fullName} #${run.number}"
              // equals pressing “Replay” + “Run” in the UI
              replay.run()
          }
      }
  }

  ```
  1. Drop above code in `Manage Jenkins ▶ Script Console` or embed it in an administravtive job
  2. you can add guards `(if (run.result == Result.SUCCESS) etc.) so only healthy pipelines are replayed
  3. If anything throws, the stack-trace appears in the console output and will not disrupt other jobs in the loop

## When to use which

Goal|Recommended method
---|---
Simply make the last successful buildrun again|Rebuild plugin or plain `/build`
Re-exercise exact Groovy CPS snapshot after a core/plugin upgrade|Replay via HTTP or Groovy
Mass-replay dozen of jobs as a smoke test|Groovy loop in Script Console (fastest, no network latency)

## A practical workflow

1. Upgrade core + plugin on container
2. Clone production and bind-mount the same `$JENKINS_HOME` into an isolated container
3. Groovy "replay storm" from Script Console (or the HTTP loop)
4. Watch Stage View - every red stage is a hard regression; every grep "missing plugin: xyz" means you pinned something wrong
5. Only after the replay storm is green do you bother with heavier tests (unit, jenkinsfile-runner, full E2E)

For most teams that maintain doezens or hundreds of Pipelines, step 3 surfaces 80 ~ 90% of breaking changes in under ten minutes and cost nothing but CPU

## How to make replay even more reliable

* `Record a "golden" build ID for each critical job`, and replay exactly that one so you know snapshot is good
* `Add guards` in the Groovy loop:
  ```groovy
  if (run.result == hudson.model.Result.SUCCESS) { replay.run() }
  ```

## Handing on the fly Jenkins jobs
### Grab script for on-the-fly jobs
Every workflow run stores the Groovy text that acutally executed, even if jobs were generated programmatically. Run this groovy script on Jenkins nodes that need to upgrade, and you will have a directory full of point-in-time Jenkinsfiles

```groovy
// Groovy console snippet: dump each job’s last script to disk
import jenkins.model.*
import org.jenkinsci.plugins.workflow.job.*
import org.apache.commons.io.IOUtils

def outDir = new File("/var/tmp/pipeline-snapshots")
outDir.mkdirs()

Jenkins.instance.getAllItems(WorkflowJob).each { job ->
    def run = job.getLastSuccessfulBuild()
    if (run) {
        def scriptText = run.getAction(org.jenkinsci.plugins.workflow.cps.replay.ReplayAction)
                           ?.originalScript
        if (scriptText) {
            def f = new File(outDir, "${job.fullName.replace('/', '_')}.jenkinsfile")
            IOUtils.write(scriptText, new FileOutputStream(f), "UTF-8")
            println "Saved ${f.path}"
        }
    }
}

```

### Create "snapshot" replay storm in the staging controller
* Clone production `$JENKINS_HOME` into a disposable container
* Upgrade core + plugins
* Copy snapshot directory into container where its path can be read by Groovy
* Run this console script to fire them off:
  ```groovy
  import jenkins.model.*
  import org.jenkinsci.plugins.workflow.job.*
  import org.jenkinsci.plugins.workflow.cps.replay.ReplayAction
  import groovy.io.FileType

  def snapDir = new File("/var/tmp/pipeline-snapshots")

  snapDir.eachFileMatch(FileType.FILES, ~/.*\.jenkinsfile/) { file ->
      def jobName = file.name.replace('_', '/').replace('.jenkinsfile','')
      def job = Jenkins.instance.getItemByFullName(jobName, WorkflowJob)
      if (job) {
          def run = job.getLastCompletedBuild()
          if (run) {
              def replay = run.getAction(ReplayAction)
              if (replay) {
                  println "Replaying ${job.fullName}"
                  replay.run(file.text)      // uses the captured script
              }
          }
      }
  }
  ```

### Difference among Rebuild/Retry, Restart from Stage, and Replay

Action|Behaviour
---|---
Rebuild/Retry plugin|Re-queues the job with current Jenkinsfile from job config an lets you tweak parameters
Restart from Stage|Resumes a failed Declarative run from the selected stage, reusing checkpoints; sipped stages are not rerun
Replay|Rns the entire pipeline uisng the captured Groovy script and identical parameters

## How to force a Replay-build to fail "safely"
We don't need to execute a full pipeline to see if it is compabile with new core + plugins. Here are some methods that can fail safely:

Level|How to inject the failure|When it trips|Why people choose it
---|---|---|---
Jenkinsfile|Ad `error 'Simulated failure for upgrade-smoe-test'` as the first step in the deploy stage (garded by an environment flag)|During replay, right after stage banner prints|Deterministic - never reaches the real deployment comman
Replay submission|hen you POST the script to `/replay/submit`, append the line error('sim fail') to the copy you send|immediately - the build aborts in Stage/node 0 before SCM checkout|No permanet ncode change; works even for auto-generated jobs
Environment gate|In Jenkinsfile add `when { expression { env.DRY_RUN == '1' } }` then call `error` 'Driy-run stop'. Set DRY_RUN=1 on the staging controller|Only on the cloned test controller|Let normaol prod builds keep working while every smoke-test run fails on purpose
Credentials sabotage|Bind a fake kubeconfig/API token to the deploy step so the step itself errors out(401,403...)|When the deployment plugin tries to connect|No code edits; good if you can't touch the pipeline
Agent unavailability|Remove/disconnect the node label used by deployment;stage fails with "NO node available..."|Before the deploy workspace even allocates|Zero code or credential changes; purly on controller-side
Groovy script console|Loop over lastSuccessfulbuilds and do: `replay.run('error("fail")\n + replay.originalScript)|Right after deserialization|One-shot smoke-test without touching any job config or env vars

### Code snippets

1. Inline "error" guard (simplest and most explicit)
    ```groovy
       pipeline {
         agent any
         stages {
           ...
           stage('Deploy to PROD') {
             when {
               expression { env.UPGRADE_SMOKE_TEST == '1' }
             }
             steps {
               error 'Simulated failure – verifying controller upgrade'
             }
           }
           ...
         }
       }
    ```
    * Set `UPGRADE_SMOKE_TEST=1` only on the cloned controller used for upgrade testings
    * On production the variable is unset, so deployment stage runs normally
2. Modify the script as you submit a Replay
   If you drive Replay via cURL:
    ```bash
    base="$JENKINS_URL/job/foo/123/replay"
    orig=$(curl -s -u $U:$T "$base/pipelineScript")

    payload="$(printf "error('simulated fail')\n%s" "$orig")"

    CRUMB=$(curl -s -u $U:$T "$JENKINS_URL/crumbIssuer/api/xml?xpath=concat(//crumbRequestField,\":\",//crumb)")

    curl -s -u $U:$T -H "$CRUMB" -X POST \
         --data-urlencode "script=$payload" \
         --data "Submit=Run" \
         "$base/submit"
    ```
    _The original Jenkinsfile is untoched;_ only this one replay-run fails
3. Controller-side mass-replay with injected failure
   ```groovy
      import jenkins.model.*
      import org.jenkinsci.plugins.workflow.job.*
      import org.jenkinsci.plugins.workflow.cps.replay.ReplayAction

      Jenkins.instance.getAllItems(WorkflowJob).each { job ->
          def run = job.getLastSuccessfulBuild()
          def act = run?.getAction(ReplayAction)
          if (act) {
              def script = "error('upgrade smoke fail')\n${act.originalScript}"
              println "Replaying ${job.fullName} – expected fail"
              act.run(script)   // creates build #N+1 that stops immediately
          }
      }
   ```
   Drop that in Manage Jenkins ▶ Script Console on the cloned controller.
   Every job gets a new buil that aborts right away, so you can watch for:
    * Class-path/deserialization errors → will appear before your injected `error` if the upgrade broke something
    * Side effects → there are none, because the `error` statement short-circuits the deployment logic
4. Credential munging (no code changes at all)
   * Copy the controller
   * Replace prod secrets ith dummy credentials that deliberately fail authentication (e.g., invalid Azure keys)
   * Replay real builds; deploy stages exit non-zero on login failures
     You still see the full step stack (so plugin class-loading is exercise) but nothing reaches prod endpoints

## Replay powershell step
In order to replay powershell from Linux Jenkins container, below are the practical ways teams keep their upgrade-test loop fast without spinning up a full production-grade Windows fleet:

| Option|What you test|What you still need|
|---|---|---|
| **A. Add one lightweight Windows agent** (VM, laptop, or Win-container)   | ✔ Controller ↔ agent remoting after the upgrade<br>✔ `powershell`, `bat`, `pwsh` step wiring<br>✔ Any plugin that shells out to MSBuild, NuGet, etc. | *One* throw-away Windows box/VM (can even be your own workstation).<br>Label it `win-smoke` (or whatever your Pipelines ask for) and point it at the test controller with `java -jar agent.jar …`.                                                           |
| **B. Convert jobs to `pwsh` step on the built-in Linux executor**         | ✔ Most scripts that are pure PowerShell (no COM, no Windows-only paths).<br>✔ Plugin class-loading, CPS deserialisation.                             | Install **PowerShell 7** in the test-controller container and make sure the **PowerShell plugin ≥2.0** is installed; then mass-replace `powershell` with `pwsh` when you inject the script for replay.<br>*Good if your scripts are already cross-platform.* |
| **C. Force-fail before any Windows step** (env flag + `error 'sim fail'`) | ✔ Pipeline loads, deserialises, allocates a node.<br>✔ Missing-plugin / symbol problems still surface.| No Windows agent at all—but you **won’t** exercise the PowerShell plugin itself. Suitable when you only care about controller/plugin breakage, not the Windows runtime.|

## Quick recipe -- one disposable Windows agent
1. Keep the upgraded controller in a Linux container
2. Spin up a temporary Windows vm
3. Connect it as an inbound agent
    ```powershell
    $controller = 'http://host.docker.internal:8081'   # or 127.0.0.1 if VM bridge
    Invoke-WebRequest "$controller/jnlpJars/agent.jar" -OutFile agent.jar
    java -jar agent.jar `
         -jnlpUrl "$controller/computer/win-smoke-test/jenkins-agent.jnlp" `
         -secret [copy-paste secret] `
         -workDir C:\jenkins

4. Label the node `win` (or whatever your production jobs use)
5. Run the replay storm exactly as described earlier. Every job that needs Windos lands on this one stub agent; the deployment stage still exists immediately with `error '💥 simulated fail'

## Zero Windows infrastructure? Use `pwsh` instead of powershell
if your script are plain Poershell (no sc.exe no registry tweaks, no Windows path assumptions), switch to the cross-platform `pwsh` step for the smoke test:
```groovy
stage('Unit tests') {
  steps {
    pwsh '''
      $ErrorActionPreference = "Stop"
      ./build.ps1 -test
    '''
  }
}
```
* Install Powershell 7 inside the Linux controller container
* In the replay-injection loop, add a regex that rewrite powershell ''' → pwsh '''

This will gives you a single-container test harness - but only if your scripts truly don't rely on Windows-only features

## If you only care about controller/plugin wiring
Keep the Linux-only setup and inject error() before the first Windows stage:
```groovy
def injected = script.replaceFirst(/stage\\s*\\(['"]Deploy.*?\\)/,
    "stage('Deploy – skipped in smoke test') {\n  steps { error 'Sim fail' }\n}\n\$0")

```
The build stops before hitting `powershell`, yet you still exercises:
 * CPS deserialisation
 * All non-Windows plugins
 * Job/Build APIs
This is great for a five-minute sanity check, but it can't tell you if the new Powshell plugin jar sudeenly fails to launch
