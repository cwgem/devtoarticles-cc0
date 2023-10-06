{%- # TOC start (generated with https://github.com/derlin/bitdowntoc) -%}

- [Execution Entry](#execution-entry)
- [The Listener](#the-listener)
- [Runner Configuration](#runner-configuration)
- [Message Listener](#message-listener)
- [Job Worker Dispatch](#job-worker-dispatch)

{%- # TOC end -%}

When running code in GitHub Actions, one piece of information that is mostly abstracted away is **how** the code is being run. Everything from authentication to the actual job execution itself. Source code of the runner which orchestrates the running of code is available in a [GitHub repository](https://github.com/actions/runner). It's primarily written in the C# programming language which is portable among several platforms through the openly available dotNET runtime. This article will look at what happens before all the log output you see on the GitHub Actions UI.

## Execution Entry

Looking around the repository there's a [run.sh](https://github.com/actions/runner/blob/f57ecd8e3c618e1723cdf02565e5f4da188776a4/src/Misc/layoutroot/run.sh) and [run.cmd](https://github.com/actions/runner/blob/f57ecd8e3c618e1723cdf02565e5f4da188776a4/src/Misc/layoutroot/run.cmd) which both are calling a `run-helper.sh` or `run-helper.cmd` in a loop format:

```bash
run() {
    # run the helper process which keep the listener alive
    while :;
    do
        cp -f "$DIR"/run-helper.sh.template "$DIR"/run-helper.sh
        "$DIR"/run-helper.sh $*
        returnCode=$?
        if [[ $returnCode -eq 2 ]]; then
            echo "Restarting runner..."
        else
            echo "Exiting runner..."
            exit 0
        fi
    done
}

```

```powershell
:launch_helper
copy "%~dp0run-helper.cmd.template" "%~dp0run-helper.cmd" /Y
call "%~dp0run-helper.cmd" %*
  
if %ERRORLEVEL% EQU 1 (
  echo "Restarting runner..."
  goto :launch_helper
) else (  
  echo "Exiting runner..."
  exit /b 0
)
```

The helpers that are called are themselves template files that get renamed from [run-helper.cmd.template](https://github.com/actions/runner/blob/f57ecd8e3c618e1723cdf02565e5f4da188776a4/src/Misc/layoutroot/run-helper.cmd.template) and [run-helper.sh.template](https://github.com/actions/runner/blob/f57ecd8e3c618e1723cdf02565e5f4da188776a4/src/Misc/layoutroot/run-helper.sh.template). Both of these will run a `Runner.Listener` executable:

```bash
"$DIR"/bin/Runner.Listener run $*
```

```powershell
"%~dp0\bin\Runner.Listener.exe" run %*
```

## The Listener

The `Runner.Listener` has an entry point of [Program.cs](https://github.com/actions/runner/blob/f57ecd8e3c618e1723cdf02565e5f4da188776a4/src/Runner.Listener/Program.cs) in the source code. At first entry an [environment is loaded](https://github.com/actions/runner/blob/f57ecd8e3c618e1723cdf02565e5f4da188776a4/src/Runner.Listener/Program.cs#L155) via this method:

```csharp
private static void LoadAndSetEnv()
{
    var binDir = Path.GetDirectoryName(Assembly.GetEntryAssembly().Location);
    var rootDir = new DirectoryInfo(binDir).Parent.FullName;
    string envFile = Path.Combine(rootDir, ".env");
    if (File.Exists(envFile))
    {
        var envContents = File.ReadAllLines(envFile);
        foreach (var env in envContents)
        {
            if (!string.IsNullOrEmpty(env))
            {
                var separatorIndex = env.IndexOf('=');
                if (separatorIndex > 0)
                {
                    string envKey = env.Substring(0, separatorIndex);
                    string envValue = null;
                    if (env.Length > separatorIndex + 1)
                    {
                        envValue = env.Substring(separatorIndex + 1);
                    }

                    Environment.SetEnvironmentVariable(envKey, envValue);
                }
            }
        }
    }
}
```

This will load an `.env` file and attach anything in it to the program's environment. You can see generation of such a file via [env.sh](https://github.com/actions/runner/blob/f57ecd8e3c618e1723cdf02565e5f4da188776a4/src/Misc/layoutroot/env.sh) which pulls in several system and language environment variables. Once that's done there are various system level checks and command validation. This will then pass [off to a runner](https://github.com/actions/runner/blob/f57ecd8e3c618e1723cdf02565e5f4da188776a4/src/Runner.Listener/Program.cs#L121) using an asynchronous (which a majority of the codebase is) task:

```csharp
IRunner runner = context.GetService<IRunner>();
try
{
    var returnCode = await runner.ExecuteCommand(command);
    trace.Info($"Runner execution has finished with return code {returnCode}");
    return returnCode;
}
```

## Runner Configuration

The runner itself lives in [Runner.cs](https://github.com/actions/runner/blob/f57ecd8e3c618e1723cdf02565e5f4da188776a4/src/Runner.Listener/Runner.cs). The actual runner command has several subcommands:

- Help: Print usage information
- Version: Print runner version information
- Commit: Print commit hash the runner was compiled from
- Check: Ensures GitHub connectivity
- Configure: Configure the GitHub runner for first time use
- Remove: Essentially a runner uninstall
- Run: Execute the runner

Now looking at [the code](https://github.com/actions/runner/blob/f57ecd8e3c618e1723cdf02565e5f4da188776a4/src/Runner.Listener/Runner.cs#L243) the runner won't even initiate if it hasn't been configured:

```csharp
if (command.Run) // this line is current break machine provisioner.
{
    // Error if runner not configured.
    if (!configManager.IsConfigured())
    {
        _term.WriteError("Runner is not configured.");
        PrintUsage(command);
        return Constants.Runner.ReturnCode.TerminatedError;
    }
```

The configuration itself is [referenced as a configManager](https://github.com/actions/runner/blob/f57ecd8e3c618e1723cdf02565e5f4da188776a4/src/Runner.Listener/Runner.cs#L126):

```csharp
try
{
    await configManager.ConfigureAsync(command);
    return Constants.Runner.ReturnCode.Success;
}
```

The definition of `configManager` exists in the file [ConfigurationManager.cs](https://github.com/actions/runner/blob/f57ecd8e3c618e1723cdf02565e5f4da188776a4/src/Runner.Listener/Configuration/ConfigurationManager.cs). This file contains various configuration options to setup along with [some fancy ascii art](https://github.com/actions/runner/blob/f57ecd8e3c618e1723cdf02565e5f4da188776a4/src/Runner.Listener/Configuration/ConfigurationManager.cs#L69):

```csharp
            _term.WriteLine();
            _term.WriteLine("--------------------------------------------------------------------------------");
            _term.WriteLine("|        ____ _ _   _   _       _          _        _   _                      |");
            _term.WriteLine("|       / ___(_) |_| | | |_   _| |__      / \\   ___| |_(_) ___  _ __  ___      |");
            _term.WriteLine("|      | |  _| | __| |_| | | | | '_ \\    / _ \\ / __| __| |/ _ \\| '_ \\/ __|     |");
            _term.WriteLine("|      | |_| | | |_|  _  | |_| | |_) |  / ___ \\ (__| |_| | (_) | | | \\__ \\     |");
            _term.WriteLine("|       \\____|_|\\__|_| |_|\\__,_|_.__/  /_/   \\_\\___|\\__|_|\\___/|_| |_|___/     |");
            _term.WriteLine("|                                                                              |");
            _term.Write("|                       ");
            _term.Write("Self-hosted runner registration", ConsoleColor.Cyan);
            _term.WriteLine("                        |");
            _term.WriteLine("|                                                                              |");
            _term.WriteLine("--------------------------------------------------------------------------------");
```

The core configuration occurs with the [runner registration](https://github.com/actions/runner/blob/f57ecd8e3c618e1723cdf02565e5f4da188776a4/src/Runner.Listener/Configuration/ConfigurationManager.cs#L135):

```csharp
else
{
    runnerSettings.GitHubUrl = inputUrl;
    registerToken = await GetRunnerTokenAsync(command, inputUrl, "registration");
    GitHubAuthResult authResult = await GetTenantCredential(inputUrl, registerToken, Constants.RunnerEvent.Register);
    runnerSettings.ServerUrl = authResult.TenantUrl;
    runnerSettings.UseV2Flow = authResult.UseV2Flow;
    Trace.Info($"Using V2 flow: {runnerSettings.UseV2Flow}");
    creds = authResult.ToVssCredentials();
    Trace.Info("cred retrieved via GitHub auth");
}
```

[GetRunnerTokenAsync](https://github.com/actions/runner/blob/f57ecd8e3c618e1723cdf02565e5f4da188776a4/src/Runner.Listener/Configuration/ConfigurationManager.cs#L643) will either use a GitHub Personal Access Token or the value of the `--token` argument to generate a token used during runner registration:

```csharp
private async Task<string> GetRunnerTokenAsync(CommandSettings command, string githubUrl, string tokenType)
{
    var githubPAT = command.GetGitHubPersonalAccessToken();
    var runnerToken = string.Empty;
    if (!string.IsNullOrEmpty(githubPAT))
    {
        Trace.Info($"Retriving runner {tokenType} token using GitHub PAT.");
        var jitToken = await GetJITRunnerTokenAsync(githubUrl, githubPAT, tokenType);
        Trace.Info($"Retrived runner {tokenType} token is good to {jitToken.ExpiresAt}.");
        HostContext.SecretMasker.AddValue(jitToken.Token);
        runnerToken = jitToken.Token;
    }

    if (string.IsNullOrEmpty(runnerToken))
    {
        if (string.Equals("registration", tokenType, StringComparison.OrdinalIgnoreCase))
        {
            runnerToken = command.GetRunnerRegisterToken();
        }
        else
        {
            runnerToken = command.GetRunnerDeletionToken();
        }
    }

    return runnerToken;
}
```

After that [GetTenantCredential](https://github.com/actions/runner/blob/f57ecd8e3c618e1723cdf02565e5f4da188776a4/src/Runner.Listener/Configuration/ConfigurationManager.cs#L758) will register the runner to the appropriate URL depending on if it's public GitHub or GitHub Hosted:

```csharp
if (UrlUtil.IsHostedServer(gitHubUrlBuilder))
{
    githubApiUrl = $"{gitHubUrlBuilder.Scheme}://api.{gitHubUrlBuilder.Host}/actions/runner-registration";
}
else
{
    githubApiUrl = $"{gitHubUrlBuilder.Scheme}://{gitHubUrlBuilder.Host}/api/v3/actions/runner-registration";
}
```

Then the credentials are finally converted to an OAuth2 format via [CredentialManager.cs](https://github.com/actions/runner/blob/f57ecd8e3c618e1723cdf02565e5f4da188776a4/src/Runner.Listener/Configuration/CredentialManager.cs):

```csharp
public VssCredentials ToVssCredentials()
{
    ArgUtil.NotNullOrEmpty(TokenSchema, nameof(TokenSchema));
    ArgUtil.NotNullOrEmpty(Token, nameof(Token));

    if (string.Equals(TokenSchema, "OAuthAccessToken", StringComparison.OrdinalIgnoreCase))
    {
        return new VssCredentials(new VssOAuthAccessTokenCredential(Token), CredentialPromptType.DoNotPrompt);
    }
    else
    {
        throw new NotSupportedException($"Not supported token schema: {TokenSchema}");
    }
}
```

As GitHub utilizes Azure services, credentials are converted to Vss format for future authorization. Once this is done the [connection is validated](https://github.com/actions/runner/blob/main/src/Runner.Listener/Configuration/ConfigurationManager.cs#L165) with our new credentials:

```csharp
// Validate can connect.
await _runnerServer.ConnectAsync(new Uri(runnerSettings.ServerUrl), creds);
```

During the registration process an RSA key will be [created as well](https://github.com/actions/runner/blob/f57ecd8e3c618e1723cdf02565e5f4da188776a4/src/Runner.Listener/Configuration/ConfigurationManager.cs#L182):

```csharp
RSAParameters publicKey;
var keyManager = HostContext.GetService<IRSAKeyManager>();
string publicKeyXML;
using (var rsa = keyManager.CreateKey())
{
    publicKey = rsa.ExportParameters(false);
    publicKeyXML = rsa.ToXmlString(includePrivateParameters: false);
}
```

This is used for both encryption and decryption of certain GitHub Actions messages (on both client and server side). Finally the agent can actually be added via [RunnerDotcomServer.cs](https://github.com/actions/runner/blob/f57ecd8e3c618e1723cdf02565e5f4da188776a4/src/Runner.Common/RunnerDotcomServer.cs#L142):

```csharp
var gitHubUrlBuilder = new UriBuilder(githubUrl);
var path = gitHubUrlBuilder.Path.Split('/', '\\', StringSplitOptions.RemoveEmptyEntries);
string githubApiUrl;
if (UrlUtil.IsHostedServer(gitHubUrlBuilder))
{
    githubApiUrl = $"{gitHubUrlBuilder.Scheme}://api.{gitHubUrlBuilder.Host}/actions/runners/register";
}
else
{
    githubApiUrl = $"{gitHubUrlBuilder.Scheme}://{gitHubUrlBuilder.Host}/api/v3/actions/runners/register";
}

var bodyObject = new Dictionary<string, Object>()
        {
            {"url", githubUrl},
            {"group_id", runnerGroupId},
            {"name", agent.Name},
            {"version", agent.Version},
            {"updates_disabled", agent.DisableUpdate},
            {"ephemeral", agent.Ephemeral},
            {"labels", agent.Labels},
            {"public_key", publicKey},
        };
```

The reason why there are two registrations is that the first ensures proper org/repository access and this actually registers the runner to the appropriate location.

## Message Listener

Once the configuration has been completed the runner is now able to execute. This is handled via the a [RunAsync call](https://github.com/actions/runner/blob/f57ecd8e3c618e1723cdf02565e5f4da188776a4/src/Runner.Listener/Runner.cs#L283). The first notable thing this does is [create a listener](https://github.com/actions/runner/blob/f57ecd8e3c618e1723cdf02565e5f4da188776a4/src/Runner.Listener/Runner.cs#L360) for messages from GitHub:

```csharp
Trace.Info(nameof(RunAsync));
_listener = GetMesageListener(settings);
if (!await _listener.CreateSessionAsync(HostContext.RunnerShutdownToken))
{
    return Constants.Runner.ReturnCode.TerminatedError;
}
```

[GetMesageListener](https://github.com/actions/runner/blob/f57ecd8e3c618e1723cdf02565e5f4da188776a4/src/Runner.Listener/Runner.cs#L342) (No that's not a typo) obtains a message listener to utilize for this purpose:

```csharp
private IMessageListener GetMesageListener(RunnerSettings settings)
{
    if (settings.UseV2Flow)
    {
        Trace.Info($"Using BrokerMessageListener");
        var brokerListener = new BrokerMessageListener();
        brokerListener.Initialize(HostContext);
        return brokerListener;
    }

    return HostContext.GetService<IMessageListener>();
}
```

In testing I found the `UseV2Flow` isn't reached for my setup and code from [MessageListener.cs](https://github.com/actions/runner/blob/f57ecd8e3c618e1723cdf02565e5f4da188776a4/src/Runner.Listener/MessageListener.cs) was called. Once initialized the `MessageListener` will create a session with GitHub via the settings it obtained during registration. These settings look something like this:

```json
{
  "AgentId": 2,
  "AgentName": "MyAgent",
  "PoolId": 1,
  "PoolName": "Default",
  "ServerUrl": "https://pipelinesghubeus3.actions.githubusercontent.com/[randomstring]/",
  "GitHubUrl": "https://github.com/org/repo",
  "WorkFolder": "_work"
}
```

The URL itself is specialized for GitHub actions and not part of the standard GitHub API URL structure. After [loading credentials](https://github.com/actions/runner/blob/f57ecd8e3c618e1723cdf02565e5f4da188776a4/src/Runner.Listener/MessageListener.cs#L64) to authenticate [session information](https://github.com/actions/runner/blob/f57ecd8e3c618e1723cdf02565e5f4da188776a4/src/Runner.Listener/MessageListener.cs#L68) is created:

```csharp
var agent = new TaskAgentReference
{
    Id = _settings.AgentId,
    Name = _settings.AgentName,
    Version = BuildConstants.RunnerPackage.Version,
    OSDescription = RuntimeInformation.OSDescription,
};
string sessionName = $"{Environment.MachineName ?? "RUNNER"}";
var taskAgentSession = new TaskAgentSession(sessionName, agent);
```

This gives some basic version and OS information for the GitHub API to identify the agent. Now an [initial connection needs to be established](https://github.com/actions/runner/blob/f57ecd8e3c618e1723cdf02565e5f4da188776a4/src/Runner.Listener/MessageListener.cs#L88) with the server indicated by `ServerUrl` in the settings:

```csharp
await _runnerServer.ConnectAsync(new Uri(serverUrl), creds);
```

Once the connection has been established a [session can be created](https://github.com/actions/runner/blob/f57ecd8e3c618e1723cdf02565e5f4da188776a4/src/Sdk/DTGenerated/Generated/TaskAgentHttpClientBase.cs#L726):

```csharp
public virtual Task<TaskAgentSession> CreateAgentSessionAsync(
    int poolId,
    TaskAgentSession session,
    object userState = null,
    CancellationToken cancellationToken = default)
{
    HttpMethod httpMethod = new HttpMethod("POST");
    Guid locationId = new Guid("134e239e-2df3-4794-a6f6-24f1f19ec8dc");
    object routeValues = new { poolId = poolId };
    HttpContent content = new ObjectContent<TaskAgentSession>(session, new VssJsonMediaTypeFormatter(true));

    return SendAsync<TaskAgentSession>(
        httpMethod,
        locationId,
        routeValues: routeValues,
        version: new ApiResourceVersion(5.1, 1),
        userState: userState,
        cancellationToken: cancellationToken,
        content: content);
}
```

This simply takes the task session information and bundles it along with a few more attributes which then gets executed as an API call. The actual call itself is run off of [SendAsync](https://github.com/actions/runner/blob/main/src/Sdk/WebApi/WebApi/VssHttpClientBase.cs#L813) which itself calls a wrapped `System.Net.Http` [HttpClient](https://learn.microsoft.com/en-us/dotnet/api/system.net.http.httpclient?view=net-7.0)'s [SendAsync method](https://learn.microsoft.com/en-us/dotnet/api/system.net.http.httpclient.sendasync?view=net-7.0). This call pattern occurs quite frequently for various calls to GitHub's APIs. Session creation will also return an `encryptionKey` as one of the values to decrypt later messages sent by the GitHub API. Once the session is established the runner [retrieves the first message](https://github.com/actions/runner/blob/f57ecd8e3c618e1723cdf02565e5f4da188776a4/src/Runner.Listener/Runner.cs#L397) from the message listener. Before looking at this though I'd like to take a quick detour at an interesting piece of functionality: cancellation tokens. You can [seen an example](https://github.com/actions/runner/blob/f57ecd8e3c618e1723cdf02565e5f4da188776a4/src/Runner.Listener/Runner.cs#L373) used with the message queue:

```csharp
CancellationTokenSource messageQueueLoopTokenSource = CancellationTokenSource.CreateLinkedTokenSource(HostContext.RunnerShutdownToken);
```

A [CancellationTokenSource](https://learn.microsoft.com/en-us/dotnet/api/system.threading.cancellationtokensource?view=net-7.0) is a dotnet concept which essentially creates a [stop flag](https://github.com/actions/runner/blob/f57ecd8e3c618e1723cdf02565e5f4da188776a4/src/Runner.Listener/Runner.cs#L391) for loop/thread type constructs:

```csharp
while (!HostContext.RunnerShutdownToken.IsCancellationRequested)
```

This means that every part of the message listener which references this token will stop execution when reaching the checked value if its `Cancel()` method is called. Utilizing this can help prevent an abnormal amount of exceptions bubbling up from underlying sources. [CreateLinkedTokenSource()](https://learn.microsoft.com/en-us/dotnet/api/system.threading.cancellationtokensource.createlinkedtokensource?view=net-7.0#system-threading-cancellationtokensource-createlinkedtokensource(system-threading-cancellationtoken)) allows for linking cancellation to other cancellation tokens. That means if a cancellation initiates at the system shutdown level it will also cancel message listener related executions using the token. This token feature is especially useful in the case where a user cancels a GitHub Action workflow/job which means everything on the runner side related to it needs to shut down.

Once the queue is entered the runner receives a message from the message listener via a call to [GetNextMessageAsync](https://github.com/actions/runner/blob/f57ecd8e3c618e1723cdf02565e5f4da188776a4/src/Runner.Listener/Runner.cs#L397). The message itself is encrypted via a combination of AES and RSA with session scope values. RSA key in this case is the one generated and registered with GitHub during the runner configuration process. If you wish to dive into it further, I put together a [GitHub Actions Message Decrypt Tool](https://github.com/cwgem/GitHubActionsMessageDecrypt) which can be used to decrypt message contents from network traffic analysis. The basic format of this message is:

- `fileTable`: list of relevant files, generally pointing to workflow YAML files
- `mask`: values to mask from log outputs based on regex
- `steps`: action steps for the job
- `variables`: system information, along with features such as [GITHUB_TOKEN](https://docs.github.com/en/actions/security-guides/automatic-token-authentication) value and permissions
- `messageType`: the type of message, with `PipelineAgentJobRequest` being the one that indicates a job needs to be run
- `plan`: related to task orchestration such as console/job status updates
- `timeline`: timeline of task status transitions
- `jobId`: ID of the job
- `jobDisplayName`: the name of the job as it's defined under the `jobs` toplevel key
- `jobName`: name of the job as declared as `name:` under each toplevel job key, set to `__default` if there isn't one defined
- `requestId`: job request ID, used in a later call
- `lockedUntil`: doesn't really have any meaningful value
- `resources`: various service endpoints and related information
- `contextData`: various context objects including [github](https://docs.github.com/en/actions/learn-github-actions/contexts#github-context), [inputs](https://docs.github.com/en/actions/learn-github-actions/contexts#inputs-context), [vars](https://docs.github.com/en/actions/learn-github-actions/contexts#vars-context), [needs](https://docs.github.com/en/actions/learn-github-actions/contexts#needs-context), [strategy](https://docs.github.com/en/actions/learn-github-actions/contexts#strategy-context), [matrix](https://docs.github.com/en/actions/learn-github-actions/contexts#matrix-context), and some [feature flags](https://github.blog/2021-04-27-ship-code-faster-safer-feature-flags/) if available

The full data structure of a message can also be found in [AgentJobRequestMessage.cs](https://github.com/actions/runner/blob/f57ecd8e3c618e1723cdf02565e5f4da188776a4/src/Sdk/DTPipelines/Pipelines/AgentJobRequestMessage.cs). Now that the message is received it's time to handle the job. Note that each entry under `jobs:` in the workflow YAML receives its own unique message and is handled separately. The message enters the job dispatch workflow as a [Run() method invocation](https://github.com/actions/runner/blob/f57ecd8e3c618e1723cdf02565e5f4da188776a4/src/Runner.Listener/Runner.cs#L517):

```csharp
Trace.Info($"Received job message of length {message.Body.Length} from service, with hash '{IOUtil.GetSha256Hash(message.Body)}'");
var jobMessage = StringUtil.ConvertFromJson<Pipelines.AgentJobRequestMessage>(message.Body);
jobDispatcher.Run(jobMessage, runOnce);
```

## Job Worker Dispatch

Job dispatch is handled as part of [JobDispatcher.cs](https://github.com/actions/runner/blob/f57ecd8e3c618e1723cdf02565e5f4da188776a4/src/Runner.Listener/JobDispatcher.cs). The `Run` method referenced actually trickles down the chain to end up at [RunAsync](https://github.com/actions/runner/blob/f57ecd8e3c618e1723cdf02565e5f4da188776a4/src/Runner.Listener/JobDispatcher.cs#L349). [ProcessChannel.cs](https://github.com/actions/runner/blob/main/src/Runner.Common/ProcessChannel.cs) creates a bi-directional channel between the runner and child worker process. Then the [worker process is executed](https://github.com/actions/runner/blob/f57ecd8e3c618e1723cdf02565e5f4da188776a4/src/Runner.Listener/JobDispatcher.cs#L468):

```csharp
string workerFileName = Path.Combine(assemblyDirectory, _workerProcessName);
workerProcessTask = processInvoker.ExecuteAsync(
    workingDirectory: assemblyDirectory,
    fileName: workerFileName,
    arguments: "spawnclient " + pipeHandleOut + " " + pipeHandleIn,
    environment: null,
    requireExitCodeZero: false,
    outputEncoding: null,
    killProcessOnCancel: true,
    redirectStandardIn: null,
    inheritConsoleHandler: false,
    keepStandardInOpen: false,
    highPriorityProcess: true,
    cancellationToken: workerProcessCancelTokenSource.Token);
```

The declaration for `_workerProcessName` can be found at the [start of the class definition](https://github.com/actions/runner/blob/f57ecd8e3c618e1723cdf02565e5f4da188776a4/src/Runner.Listener/JobDispatcher.cs#L49) as:

```csharp
private static readonly string _workerProcessName = $"Runner.Worker{IOUtil.ExeExtension}";
```

 Which will call `Runner.Worker` on *NIX systems and `Runner.Worker.exe` on Windows based systems. The dispatcher will then [send off the job details](https://github.com/actions/runner/blob/f57ecd8e3c618e1723cdf02565e5f4da188776a4/src/Runner.Listener/JobDispatcher.cs#L489) to the worker for processing:

 ```csharp
Trace.Info($"Send job request message to worker for job {message.JobId}.");
HostContext.WritePerfCounter($"RunnerSendingJobToWorker_{message.JobId}");
using (var csSendJobRequest = new CancellationTokenSource(_channelTimeout))
{
    await processChannel.SendAsync(
        messageType: MessageType.NewJobRequest,
        body: JsonUtility.ToString(message),
        cancellationToken: csSendJobRequest.Token);
}
 ```

 Now it's time for the public facing part of a GitHub action runner: the Runner Worker. Being a program the main entry point sits in [Program.cs](https://github.com/actions/runner/blob/f57ecd8e3c618e1723cdf02565e5f4da188776a4/src/Runner.Worker/Program.cs). After some argument and environment validation the [worker process starts](https://github.com/actions/runner/blob/f57ecd8e3c618e1723cdf02565e5f4da188776a4/src/Runner.Worker/Program.cs#L50):

 ```csharp
 // Run the worker.
return await worker.RunAsync(
    pipeIn: args[1],
    pipeOut: args[2]);
 ```

 The Worker will [wait for the message](https://github.com/actions/runner/blob/f57ecd8e3c618e1723cdf02565e5f4da188776a4/src/Runner.Worker/Program.cs#L50) from the job dispatcher before proceeding:

 ```csharp
 channel.StartClient(pipeIn, pipeOut);

// Wait for up to 30 seconds for a message from the channel.
HostContext.WritePerfCounter("WorkerWaitingForJobMessage");
Trace.Info("Waiting to receive the job message from the channel.");
WorkerMessage channelMessage;
using (var csChannelMessage = new CancellationTokenSource(_workerStartTimeout))
{
    channelMessage = await channel.ReceiveAsync(csChannelMessage.Token);
}
 ```

 This message is the unencrypted form of the message listener message with slight modification containing all details needed for the job. Then the job dispatcher will finally [send it off to the job runner](https://github.com/actions/runner/blob/f57ecd8e3c618e1723cdf02565e5f4da188776a4/src/Runner.Worker/Program.cs#L50):

 ```csharp
 Task<TaskResult> jobRunnerTask = jobRunner.RunAsync(jobMessage, jobRequestCancellationToken.Token);
 ```

 As with all tasks it has a cancellation token presented for handling exceptions and job cancellation requests. From here on is where all the public facing GitHub Actions output happens and the subject of the next installment in the series.

 ## Conclusion

 I must say this was a very enlightening experience. Seeing that the runner was developer in C# was a bit surprising. Between delegates and interfaces it did make for quite a substantial amount of back and forth between code files... I also found that GitHub Actions is essentially [Azure Pipelines](https://azure.microsoft.com/en-us/products/devops/pipelines/) and the agent looks to be basically the GitHub Agent runner. Not surprising though since consolidating codebases to save time seems like a fairly reasonable approach. I was actually planning on making this a very long article but figured there were those who wanted to know **everything** about GitHub Actions execution and those who simply wanted to focus on how job execution is handled in terms of what you see in the UI. I hope you enjoyed this article and please look forward to the next part of this series where I look over how the actual jobs are executed.